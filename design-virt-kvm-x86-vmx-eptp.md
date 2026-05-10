---
title: "Tier-3: arch/x86/kvm/vmx/vmx.c (EPTP subset) — VMX EPT-pointer construction + INVEPT semantics"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

VMX EPT-pointer (vmcs.EPT_POINTER, field 0x201A) is the per-vCPU root of the **Extended Page Table** that translates guest-physical to host-physical. KVM `construct_eptp(root_hpa)` packs root-HPA + PWL (page-walk-length, typically 4 = 5-level disabled / 5 = 5-level EPT) + memory-type (typically WB) into a 64-bit EPTP. Per-vCPU vmcs.EPT_POINTER set at vmenter; HW walks EPT on guest GPA→HPA. Per-INVEPT instruction invalidates EPT-walk-cache (per-context or all-context). Critical for: nested-paging on Intel; replaces shadow-MMU on hardware that supports EPT.

This Tier-3 covers EPTP subset of `vmx.c` (~80 lines).

### Acceptance Criteria

- [ ] AC-1: construct_eptp(root_hpa, 4): EPTP bits[5:3] = 011 (PWL=4).
- [ ] AC-2: construct_eptp with A/D-bits supported: bit[6] set.
- [ ] AC-3: construct_eptp with 5-level: bits[5:3] = 100.
- [ ] AC-4: construct_eptp memory-type WB: bits[2:0] = 110.
- [ ] AC-5: vmcs.EPT_POINTER = construct_eptp output.
- [ ] AC-6: Per-EPT remap: INVEPT_SINGLE_CONTEXT issued.
- [ ] AC-7: cpu_has_vmx_ept_5levels(): module init reports support.
- [ ] AC-8: Per-VM shared_eptp: all vCPUs vmcs.EPT_POINTER same root.
- [ ] AC-9: Per-nested L2: L0 vmcs02.EPT_POINTER ≠ L1's vmcs12.EPT_POINTER.
- [ ] AC-10: kvm-unit-tests `ept` test passes.

### Architecture

Per-vCPU VMX state:

```
struct VmxVcpu {
  ...
  ept_root_level: u8,                            // 4 or 5
  ...
}
```

Per-VM VMX state:

```
struct VmxArch {
  shared_eptp: Hpa,                              // shared per-VM EPT root
  ...
}
```

Constants:

```
const VMX_EPT_DEFAULT_MT: u64 = 0x6;             // WB
const VMX_EPT_DEFAULT_GAW: u64 = 0x3;            // PWL = 4
const VMX_EPT_AD_ENABLE_BIT: u64 = 1 << 6;
const VMX_EPT_EXECUTE_ONLY_BIT: u64 = 1 << 7;

const EPT_POINTER: u32 = 0x201A;
```

`Vmx::construct_eptp(root_hpa: Hpa, level: u8) -> u64`:
1. eptp = VMX_EPT_DEFAULT_MT.                     // bits[2:0] = 6 (WB).
2. eptp |= (level - 1) << 3.                       // bits[5:3].
3. If enable_ept_ad_bits ∧ Vmx::has_ept_ad_bits():
   - eptp |= VMX_EPT_AD_ENABLE_BIT.               // bit[6].
4. If allow_smaller_max_phys_addr ∧ Vmx::has_ept_x_only():
   - eptp |= VMX_EPT_EXECUTE_ONLY_BIT.             // bit[7].
5. eptp |= root_hpa.                                // bits[N-1:12].
6. Return eptp.

`Vmx::vmcs_set_eptp(vcpu, root_hpa)`:
1. eptp = Vmx::construct_eptp(root_hpa, vcpu.vmx.ept_root_level).
2. vmcs_write64(EPT_POINTER, eptp).

`Vmx::invept_global()`:
1. /* Per-host: invalidate all EPT-cached translations. */
2. asm_volatile ("invept %0, %1" :: "m"(0u128), "r"(VMX_INVEPT_GLOBAL)).

`Vmx::invept_context(eptp)`:
1. /* Per-EPTP-context: invalidate. */
2. desc = (eptp, 0).
3. asm_volatile ("invept %0, %1" :: "m"(desc), "r"(VMX_INVEPT_SINGLE_CONTEXT)).

`Vmx::flush_tlb_guest_eptp(vcpu)`:
1. eptp = vmcs_read64(EPT_POINTER).
2. Vmx::invept_context(eptp).

`Mmu::init_shadow_ept_mmu(vcpu, level)`:
1. Initialize shadow-EPT MMU with per-vCPU level.
2. Allocate root SPTE.
3. construct_eptp + vmcs_write64.

### Out of Scope

- KVM TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- KVM Shadow MMU (covered in `x86-mmu-shadow.md` Tier-3)
- KVM SPTE format (covered in `x86-mmu-spte.md` Tier-3)
- KVM PML (covered in `x86-pml.md` Tier-3)
- KVM PAT (covered in `x86-pat.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `construct_eptp()` | per-vCPU EPTP value-builder | `Vmx::construct_eptp` |
| `vmx->ept_root_level` | per-vCPU page-walk-level | `VmxVcpu::ept_root_level` |
| `EPT_POINTER` (vmcs field 0x201A) | per-vCPU EPT root | UAPI |
| `kvm_init_shadow_ept_mmu()` | per-vCPU shadow-EPT init | `Mmu::init_shadow_ept_mmu` |
| `vmx->shared_eptp` | per-VM shared EPTP (TDP) | `VmxArch::shared_eptp` |
| `INVEPT` instruction encoding | per-context-flush opcode | shared |
| `vmx_invept_global()` | per-host INVEPT all-contexts | `Vmx::invept_global` |
| `vmx_invept_context()` | per-VM INVEPT single-context | `Vmx::invept_context` |
| `cpu_has_vmx_ept_5levels()` | per-host 5-level-EPT capability | `VmxOps::has_ept_5levels` |
| `cpu_has_vmx_ept_ad_bits()` | per-host A/D-bit capability | `VmxOps::has_ept_ad_bits` |

### compatibility contract

REQ-1: EPT-pointer (EPTP) bit layout (64-bit):
- Bits[2:0] memory-type (0=UC, 6=WB).
- Bits[5:3] page-walk-length minus 1 (3 = 4-level, 4 = 5-level).
- Bit[6] enable-A/D-bits (Broadwell+).
- Bit[7] enforce-EPT-execute-only (Sky-Lake+).
- Bits[11:8] reserved.
- Bits[N-1:12] root-page-frame (where N = MAXPHYADDR).
- Bits[63:N] reserved (must be 0).

REQ-2: construct_eptp(root_hpa, level):
- eptp = VMX_EPT_DEFAULT_MT (6=WB).
- eptp |= (level - 1) << 3.
- If cpu_has_vmx_ept_ad_bits ∧ enable_ept_ad_bits: eptp |= VMX_EPT_AD_ENABLE_BIT.
- If cpu_has_vmx_ept_x_only_bit: eptp |= VMX_EPT_EXECUTE_ONLY_BIT (depends).
- eptp |= root_hpa.

REQ-3: Per-vCPU EPT-root level:
- 4-level (PWL=4): max-phys-addr 48 bits (256TB).
- 5-level (PWL=5): max-phys-addr 57 bits (128PB).
- Per-VM gated by host CPU capability.

REQ-4: Per-vCPU vmcs.EPT_POINTER write:
- vmcs_write64(EPT_POINTER, construct_eptp(root_hpa, level)).
- Per-vmenter HW reads EPT_POINTER for EPT-walk.

REQ-5: Per-VM TDP MMU shared-EPTP:
- All vCPUs in same VM share single EPT root (per-arch-id).
- Per-vCPU vmcs.EPT_POINTER points to same root_hpa.

REQ-6: INVEPT instruction:
- Single-context: invalidates per-EPTP cached translations.
- All-context: invalidates all EPTPs.
- Per-VM-EPT change: KVM issues INVEPT_SINGLE_CONTEXT.

REQ-7: vmx_invept_global / context:
- Per-host KVM unload: invept_global.
- Per-VM-flush-EPT: invept_context(eptp).

REQ-8: A/D-bits (EPT-A/D):
- HW maintains per-EPT-PTE Accessed + Dirty bits.
- KVM dirty-tracking uses EPT-D bit (combined with PML or rmap).

REQ-9: EPT-execute-only:
- VMX_EPT_EXECUTE_ONLY_BIT: per-EPT-PTE bits[2:0] can be 100 (X-only).
- Used for kernel-text-mapping protection.

REQ-10: Per-MMU-cache flush coordination:
- Per-EPT remap: KVM_REQ_TLB_FLUSH_GUEST → INVEPT_SINGLE.
- Per-vCPU resumes only after flush completes.

REQ-11: Per-nested L2:
- L1 may pass own EPTP via vmcs12; L0 builds shadow-EPT from L1's EPTP+vmcs12.
- L0's vmcs02.EPT_POINTER points to L0-managed EPT.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `eptp_mt_in_set` | INVARIANT | EPTP[2:0] ∈ {0=UC, 6=WB}. |
| `eptp_pwl_in_range` | INVARIANT | EPTP[5:3] ∈ {3, 4} → level ∈ {4, 5}. |
| `root_hpa_4k_aligned` | INVARIANT | root_hpa & 0xFFF == 0. |
| `eptp_reserved_bits_zero` | INVARIANT | EPTP bits[63:N] == 0 (N = MAXPHYADDR). |
| `ad_bit_iff_supported` | INVARIANT | EPTP bit[6] set ⟺ host supports + enabled. |

### Layer 2: TLA+

`virt/kvm/vmx_eptp.tla`:
- Per-vCPU EPTP build + per-VM shared root + INVEPT semantics.
- Properties:
  - `safety_eptp_consistent_within_vm` — per-vCPU vmcs.EPT_POINTER same root in TDP-MMU shared mode.
  - `safety_invept_after_remap` — per-EPT-PTE remap ⟹ INVEPT_SINGLE_CONTEXT.
  - `liveness_invept_eventually_propagates` — per-INVEPT issued ⟹ subsequent walks see new mapping.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vmx::construct_eptp` post: bits packed per Intel SDM §28.2.4 | `Vmx::construct_eptp` |
| `Vmx::vmcs_set_eptp` post: vmcs.EPT_POINTER = constructed value | `Vmx::vmcs_set_eptp` |
| `Vmx::invept_context` post: per-EPTP cached translations invalidated | `Vmx::invept_context` |
| `Mmu::init_shadow_ept_mmu` post: vmcs.EPT_POINTER points to shadow-EPT root | `Mmu::init_shadow_ept_mmu` |

### Layer 4: Verus/Creusot functional

`Per-vmenter EPT_POINTER → HW walks EPT for guest GPA→HPA → per-EPT-PTE permission/memtype enforced` semantic equivalence: per-Intel SDM §28.2 EPT spec.

### hardening

(Inherits row-1 features from `virt/kvm/x86-mmu-tdp.md` § Hardening.)

EPTP-specific reinforcement:

- **Per-EPTP MT validated as WB or UC** — defense against per-config invalid memtype.
- **Per-PWL bounded {4, 5}** — defense against per-config invalid level.
- **Per-root_hpa 4K-aligned** — defense against per-misaligned EPT walk-fault.
- **Per-AD-bit only when host supports** — defense against per-flag invalid on legacy CPU.
- **Per-EPTP reserved bits zero** — defense against per-future-bit conflict.
- **Per-INVEPT after remap** — defense against per-stale TLB-cache hit.
- **Per-VM shared_eptp consistency** — defense against per-vCPU divergent EPT view.
- **Per-VM destroy: invept_global** — defense against per-VM-leak EPT-cache.
- **Per-nested L2 separate EPTP from L1** — defense against L1 spoofing L0 EPT.
- **Per-construct_eptp called via locked path** — defense against per-eptp racing with vmenter.
- **Per-EPT-execute-only gated by host cap** — defense against per-config X-only on non-Sky-Lake.

