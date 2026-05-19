# Tier-3: arch/x86/kvm/x86.c (PDPTR subset) — KVM PAE-paging PDPTR (Page-Directory-Pointer-Table-Entry) load + validate

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-mmu-shadow.md
upstream-paths:
  - arch/x86/kvm/x86.c (load_pdptrs; see grep "load_pdptrs|pdptrs")
  - arch/x86/kvm/mmu/mmu.c (mmu.pae_root)
  - arch/x86/kvm/vmx/vmx.c (vmcs.GUEST_PDPTR0..3)
-->

## Summary

PAE (Physical Address Extension) paging in 32-bit + 64-bit non-LMA modes uses 4 PDPTRs (Page-Directory-Pointer Table Entries) loaded directly into CPU when guest CR3 written. Per-PDPTR encodes a 4KB-aligned page-directory base + present + reserved bits. KVM's `load_pdptrs` validates per-PDPTR (reserved-bits zero, address ≤ MAXPHYADDR), copies into `vcpu.arch.mmu.pdptrs[4]`, and propagates to vmcs.GUEST_PDPTR0..3 (VMX) for HW-assisted load on vmenter. Critical because in PAE mode CR3 doesn't directly index PDEs — instead points to a 32-byte PDPT in guest memory; PDPTRs cached in CPU on CR3 write.

This Tier-3 covers PDPTR handling in `arch/x86/kvm/x86.c` (~100-150 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `load_pdptrs(vcpu, cr3)` | per-vCPU PDPTR load + validate | `Vcpu::load_pdptrs` |
| `vcpu.arch.mmu.pdptrs[4]` | per-vCPU PDPTR shadow | `Mmu::pdptrs` |
| `is_pae_paging(vcpu)` | per-vCPU PAE-mode query | `Vcpu::is_pae_paging` |
| `vmx_set_guest_cr3(vcpu, cr3)` (vmx.c) | VMX vmcs.GUEST_CR3 + GUEST_PDPTR0..3 write | `VcpuVmx::set_guest_cr3` |
| `vmcs.GUEST_PDPTR0..3` | per-vCPU HW-PDPTR fields | UAPI |
| `mmu.pae_root` | per-vCPU PAE-root buffer (4 PDPTEs) | `Mmu::pae_root` |
| `kvm_set_cr3(vcpu, cr3)` (covers PAE branch) | guest CR3 write | covered in `x86-cr0-cr4.md` |
| `kvm_pdptr_read(vcpu, idx)` | per-vCPU PDPTR read | `Vcpu::pdptr_read` |
| `mmu_pdptrs_changed(vcpu)` | per-vCPU PDPTR compared | `Mmu::pdptrs_changed` |
| `is_pae(vcpu)` | per-vCPU PAE-bit (CR4.PAE) check | `Vcpu::is_pae` |

## Compatibility contract

REQ-1: PAE-paging activation:
- CR4.PAE = 1 + CR0.PG = 1 + EFER.LMA = 0 (32-bit-PAE).
- OR CR4.PAE = 1 + CR0.PG = 1 + EFER.LMA = 1 (long-mode; doesn't use PDPTRs directly).
- KVM tracks via `is_pae_paging(vcpu)`.

REQ-2: PDPT layout:
- 4 PDPTEs × 8 bytes = 32 bytes in guest memory.
- CR3 points to PDPT (must be 32-byte-aligned).

REQ-3: Per-PDPTE encoding (64-bit):
- Bit[0] P (Present).
- Bit[3] PWT.
- Bit[4] PCD.
- Bit[12..MAXPHYADDR] PD-base addr.
- Bit[63] NX (if EFER.NXE).
- Reserved bits: bits[1..2], bits[5..11].

REQ-4: `load_pdptrs(vcpu, cr3)` flow:
1. If !is_pae_paging(vcpu): return 0 (no PDPTRs needed).
2. pdpt_gpa := cr3 & ~0x1F (clear lowest 5 bits).
3. Validate pdpt_gpa within MAXPHYADDR.
4. For i in 0..4:
   - pdpte_gpa := pdpt_gpa + i * 8.
   - pdpte := kvm_read_guest_page(vcpu, pdpte_gpa, 8).
   - If !pdpte.present: continue (entry unused).
   - Validate reserved-bits zero.
   - Validate addr ≤ MAXPHYADDR.
5. Copy pdpte's into vcpu.arch.mmu.pdptrs[].
6. mark_page_dirty (per-PDPT page).
7. Vendor: vmx_set_guest_cr3 → VMWRITE GUEST_PDPTR0..3 = pdpte's.

REQ-5: Per-vCPU PDPTR shadow:
- vcpu.arch.mmu.pdptrs[4]: kept in-sync with vmcs (VMX) or vmcb (SVM).
- Used by KVM's MMU for guest-PD walk.

REQ-6: VMX vmcs.GUEST_PDPTR0..3:
- HW reads at vmenter to populate CPU's PDPTR cache.
- Updated by KVM only when CR3 changes + guest in PAE mode.

REQ-7: SVM uses similar mechanism via vmcb.save area:
- vmcb.save.cr3 + per-PDPTE in cache.
- AMD-VMSAVE/_VMLOAD includes PDPTRs.

REQ-8: Per-vCPU CR3 write triggers PDPTR re-load:
- kvm_set_cr3 → (if PAE-paging): load_pdptrs(vcpu, cr3).

REQ-9: Per-vCPU CR0 / CR4 transitions:
- CR4.PAE 0 → 1: triggers load_pdptrs.
- CR0.PG 0 → 1 with PAE: triggers load_pdptrs.

REQ-10: Per-vCPU PDPTR vs nested-paging:
- With TDP MMU: PDPTRs at CPU level still required for guest paging; KVM tracks them.
- Without TDP (shadow MMU): KVM walks guest PDPTRs via shadow paging.

REQ-11: PDPTR reserved-bit validation:
- Per-vCPU mask: reserved_bits = (1 << 1) | (1 << 2) | (0xFE << 5) | (high addr bits beyond MAXPHYADDR).
- If pdpte & reserved_bits: error → kvm_inject_gp(vcpu, 0).

REQ-12: Live migration:
- Per-vCPU PDPTRs migrated as part of vmcs/vmcb state.

## Acceptance Criteria

- [ ] AC-1: 32-bit Linux guest with PAE: kernel writes CR3; load_pdptrs validates + copies.
- [ ] AC-2: Reserved bit set in PDPTE: load_pdptrs returns false; #GP injected to guest.
- [ ] AC-3: PDPTE addr > MAXPHYADDR: load_pdptrs returns false.
- [ ] AC-4: PAE-mode entry: CR4.PAE write triggers load_pdptrs on next vmenter prep.
- [ ] AC-5: vmcs.GUEST_PDPTR0..3 reflects latest load_pdptrs result.
- [ ] AC-6: Live migration: post-migrate guest paging continues (PDPTRs preserved).
- [ ] AC-7: 64-bit long-mode guest: load_pdptrs no-op (returns 0).
- [ ] AC-8: PAE → long-mode transition: PDPTRs cleared.
- [ ] AC-9: Multi-vCPU SMP: per-vCPU PDPTRs distinct.
- [ ] AC-10: kvm-unit-tests `pae_paging` test passes.

## Architecture

`Mmu::pdptrs` per-vCPU (in vcpu.arch.mmu):

```
struct Mmu {
  ...
  pdptrs: [u64; 4],
  pae_root: KBox<[u64; 4]>,                    // for PAE-shadow MMU
  ...
}
```

`Vcpu::load_pdptrs(vcpu, cr3)`:
1. If !is_pae_paging(vcpu): return 0.
2. pdpt_gpa := cr3 & ~PAGE_MASK_5 (32-byte aligned).
3. ret := kvm_read_guest_page(vcpu.kvm, gfn_to_pfn(vcpu, pdpt_gpa >> PAGE_SHIFT), pdptes, pdpt_gpa & ~PAGE_MASK, sizeof(pdptes)).
4. If ret < 0: return 0.
5. For i in 0..4:
   - If pdptes[i] & PT_PRESENT_MASK:
     - reserved := vcpu.arch.mmu.pdptr_rsvd_bits.
     - If pdptes[i] & reserved: return 0.
6. memcpy(vcpu.arch.mmu.pdptrs, pdptes, sizeof(pdptes)).
7. Vendor: vmx_load_pdptrs (VMX) writes vmcs.GUEST_PDPTR0..3.
8. mark_page_dirty(vcpu.kvm, pdpt_gpa >> PAGE_SHIFT).
9. Return 1.

`VcpuVmx::set_guest_cr3(vcpu, cr3)`:
1. VMWRITE GUEST_CR3 = cr3.
2. If is_pae_paging(vcpu):
   - VMWRITE GUEST_PDPTR0 = vcpu.arch.mmu.pdptrs[0].
   - ... GUEST_PDPTR1..3.

`Mmu::pdptrs_changed(vcpu)` (per-vmenter):
1. If !is_pae_paging(vcpu): return false.
2. cr3 := kvm_read_cr3(vcpu).
3. Read PDPTRs from guest memory.
4. Compare to vcpu.arch.mmu.pdptrs.
5. If differ: load_pdptrs(vcpu, cr3); return true.
6. Return false.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pdptr_idx_bounded` | OOB | per-PDPTR idx ∈ [0, 3]; defense against array OOB. |
| `pdpt_alignment` | INVARIANT | pdpt_gpa 32-byte aligned. |
| `pdpte_reserved_bits_zero` | INVARIANT | per-present-PDPTE reserved bits = 0. |
| `pdpte_addr_within_maxphyaddr` | INVARIANT | per-PDPTE PD-base ≤ MAXPHYADDR. |
| `vmcs_pdptr_consistent_with_shadow` | INVARIANT | vmcs.GUEST_PDPTRn == vcpu.arch.mmu.pdptrs[n]. |

### Layer 2: TLA+

`virt/kvm/load_pdptrs.tla`:
- Per-vCPU CR3 write + PAE state.
- Properties:
  - `safety_pdptrs_loaded_iff_pae` — load_pdptrs runs iff is_pae_paging.
  - `safety_invalid_pdptr_aborts` — per-invalid PDPTE causes load_pdptrs to fail.
  - `liveness_pdptrs_eventually_visible` — post-load, vmcs reflects new PDPTRs at next vmenter.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::load_pdptrs` post: per-vCPU pdptrs[] populated from guest memory + validated | `Vcpu::load_pdptrs` |
| `VcpuVmx::set_guest_cr3` post: vmcs.GUEST_CR3 + GUEST_PDPTR0..3 updated | `VcpuVmx::set_guest_cr3` |
| `Mmu::pdptrs_changed` post: returns true iff pdptrs differ; load_pdptrs called | `Mmu::pdptrs_changed` |
| Per-vCPU PDPTRs invalidated on PAE → non-PAE transition | invariants on CR4.PAE change |

### Layer 4: Verus/Creusot functional

`Per-CR3 write in PAE: 4 PDPTEs read from guest memory + validated + cached + written to vmcs/vmcb` semantic equivalence: per-write the CPU sees PDPTRs matching guest's PDPT contents.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-mmu-shadow.md` § Hardening.)

PDPTR-specific reinforcement:

- **Per-PDPTE reserved-bits validation** — defense against guest writing invalid reserved bits causing CPU exception.
- **Per-PDPTE addr ≤ MAXPHYADDR** — defense against high-bit attack via long-mode-style addr.
- **PDPT 32-byte alignment** — defense against split-page read causing torn entry.
- **load_pdptrs on CR3/CR4.PAE/CR0.PG transitions** — defense against stale-PDPTR after mode change.
- **mark_page_dirty per-PDPT-read** — defense against mmu-notifier missing PDPT-page.
- **Per-vmenter pdptrs_changed check** — defense against guest in-place PDPT modification.
- **Per-vCPU pdptrs distinct from L2 pdptrs (nested)** — defense against cross-mode PDPTR leak.
- **vmcs.GUEST_PDPTRn writes after CR3 write** — defense against CPU vmenter consistency-check fail.
- **Per-mode PAE detection accurate** — defense against load_pdptrs running in long-mode.
- **Live-migrate PDPTRs included** — defense against post-migrate page-table inconsistency.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — PDPTR ioctl payloads (KVM_GET/SET_SREGS, KVM_GET/SET_VCPU_EVENTS) bounded.
- **PAX_KERNEXEC** — load_pdptrs path RO after init.
- **PAX_RANDKSTACK** — randomized kstack per PDPTR-load (CR3 change / vmenter).
- **PAX_REFCOUNT** — PDPT page ref saturating.
- **PAX_MEMORY_SANITIZE** — vCPU PDPTR cache zeroed on destroy / INIT.
- **PAX_UDEREF** — guest CR3 → PDPT hva validated.
- **PAX_RAP / kCFI** — load_pdptrs vendor callbacks type-checked.
- **GRKERNSEC_HIDESYM** — PDPTR raw values + PDPT PA redacted.
- **GRKERNSEC_DMESG** — invalid-PDPTR warnings rate-limited.

PDPTR-specific:

- **CAP_SYS_ADMIN on KVM_SET_SREGS** — privileged VMM only.
- **PDPTE.addr ≤ host MAXPHYADDR enforced** — closes high-bit alias.
- **L1-vs-L2 PDPTR cache separated** — defense against nested leak.

Rationale: PDPTR is the PAE root chokepoint; sanitize + UDEREF on the four-entry table prevent stale-entry carryover after CR3/CR4 mode transitions.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- Shadow MMU (covered in `x86-mmu-shadow.md` Tier-3)
- TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- CR0/CR4 (covered in `x86-cr0-cr4.md` Tier-3)
- VMX core (covered in `x86-vmx.md` Tier-3)
- SVM core (covered in `x86-svm.md` Tier-3)
- Nested-VMX PDPTRs (covered in `x86-nested-vmx.md` Tier-3)
- Implementation code
