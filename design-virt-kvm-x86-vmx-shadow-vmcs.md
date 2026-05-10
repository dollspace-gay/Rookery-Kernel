---
title: "Tier-3: arch/x86/kvm/vmx/nested.c (shadow-vmcs subset) — KVM nested-VMX shadow-VMCS for L2 VMREAD/VMWRITE acceleration"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Nested-VMX shadow-VMCS (Sky-Lake+) avoids vmexit on L2-issued VMREAD/VMWRITE by allowing HW to operate on a **shadow** VMCS (subset of L1's vmcs12 fields) directly. KVM marks read-only-fields (~30) and read-write-fields (~50) per `vmcs_shadow_fields.h`; HW handles those in-place. Per-non-shadowed fields trigger vmexit (KVM emulates). Per-vmcs01 has separate shadow_vmcs page; per-vmcs02 transitions read/write the shadow region. Per-VMPTRLD/VMCLEAR drives shadow-vmcs lifecycle. Per-module-param `enable_shadow_vmcs`. Critical for: nested-VM perf (L1 hypervisor running L2 with frequent VMREAD/VMWRITE).

This Tier-3 covers shadow-vmcs subset of `nested.c` (~250 lines).

### Acceptance Criteria

- [ ] AC-1: Boot KVM with shadow-VMCS-capable CPU + nested L1 KVM: enable_shadow_vmcs=1.
- [ ] AC-2: Per-vmcs01 SHADOW_VMCS bit set; VMREAD/VMWRITE_BITMAP populated.
- [ ] AC-3: L2 VMREAD GUEST_RIP: no vmexit (HW reads shadow).
- [ ] AC-4: L2 VMWRITE GUEST_CR0: no vmexit.
- [ ] AC-5: L2 VMWRITE non-shadowed (e.g. EPT_POINTER): vmexit; KVM emulates.
- [ ] AC-6: nested_cache_shadow_vmcs12: vmcs12 loaded from guest mem.
- [ ] AC-7: copy_vmcs02_to_shadow on vmenter: shadow updated.
- [ ] AC-8: copy_shadow_to_vmcs12 on vmexit: vmcs12 in guest mem updated.
- [ ] AC-9: enable_shadow_vmcs=0: all VMREAD/VMWRITE vmexit.
- [ ] AC-10: kvm-unit-tests `nested-shadow-vmcs` passes.

### Architecture

Per-field descriptor:

```
struct ShadowVmcsField {
  encoding: u16,                                 // VMCS-field-encoding
  offset: u16,                                   // offset in vmcs12 struct
}

const VMCS_BITMAP_PG_SIZE: usize = 4096;
```

Per-vmcs01:

```
struct LoadedVmcs {
  vmcs: PhysAddr,
  shadow_vmcs: Option<PhysAddr>,                 // shadow-vmcs page
  msr_bitmap: Box<[u8; 4096]>,
  ...
}
```

Per-VMX nested:

```
struct VmxNested {
  ...
  cached_shadow_vmcs12: Option<Box<Vmcs12>>,     // per-vCPU cache
  shadow_vmcs12_cache: GfnToHvaCache,             // per-VMPTRLD cache
  ...
}
```

`Vmx::init_shadow_fields()`:
1. /* per-CPU bitmap pages */
2. vmread_bitmap = alloc_page(); vmwrite_bitmap = alloc_page().
3. memset(vmread_bitmap, 0xff, PAGE_SIZE).      // default all intercept
4. memset(vmwrite_bitmap, 0xff, PAGE_SIZE).
5. for entry in shadow_read_only_fields:
   - clear_bit(entry.encoding, vmread_bitmap).   // HW reads directly
   - /* vmwrite stays set (RO not writable from L2) */
6. for entry in shadow_read_write_fields:
   - clear_bit(entry.encoding, vmread_bitmap).
   - clear_bit(entry.encoding, vmwrite_bitmap).

`Vmx::vcpu_create_shadow_vmcs(vmx) -> Result<()>`:
1. If !enable_shadow_vmcs: return Ok.
2. vmx.vmcs01.shadow_vmcs = alloc_vmcs().
3. vmcs_clear(vmx.vmcs01.shadow_vmcs).
4. /* Per-vmcs01 setup */
5. vmcs_write64(VMCS_LINK_POINTER, vmx.vmcs01.shadow_vmcs.phys).
6. vmcs_write64(VMREAD_BITMAP, vmread_bitmap.phys).
7. vmcs_write64(VMWRITE_BITMAP, vmwrite_bitmap.phys).
8. vmcs_set_bits(SECONDARY_VM_EXEC_CONTROL, SHADOW_VMCS).

`Vmx::disable_shadow_vmcs(vmx)`:
1. vmcs_clear_bits(SECONDARY_VM_EXEC_CONTROL, SHADOW_VMCS).
2. vmcs_write64(VMCS_LINK_POINTER, ~0ULL).

`Nested::cache_shadow_vmcs12(vcpu, vmptr)`:
1. ghc = &vmx.nested.shadow_vmcs12_cache.
2. err = kvm_gfn_to_hva_cache_init(vcpu.kvm, ghc, vmptr, VMCS12_SIZE).
3. err = kvm_read_guest_cached(ghc, vmx.nested.cached_shadow_vmcs12, VMCS12_SIZE).

`Nested::copy_vmcs02_to_shadow(vcpu)`:
1. /* For each shadow field: copy vmcs12 → shadow_vmcs. */
2. shadow_vmcs = vmx.vmcs01.shadow_vmcs.
3. for field in shadow_read_only_fields ∪ shadow_read_write_fields:
   - val = vmcs12_field_get(vmcs12, field.encoding).
   - vmcs_load(shadow_vmcs); vmcs_writeXX(field.encoding, val).

`Nested::copy_shadow_to_vmcs12(vcpu)`:
1. shadow_vmcs = vmx.vmcs01.shadow_vmcs.
2. for field in shadow_read_write_fields:
   - vmcs_load(shadow_vmcs); val = vmcs_readXX(field.encoding).
   - vmcs12_field_set(vmcs12, field.encoding, val).

`Nested::vcpu_destroy_shadow_vmcs(vmx)`:
1. If vmx.vmcs01.shadow_vmcs:
   - vmx_disable_shadow_vmcs(vmx).
   - vmcs_clear(vmx.vmcs01.shadow_vmcs).
   - free_vmcs(vmx.vmcs01.shadow_vmcs).
   - vmx.vmcs01.shadow_vmcs = NULL.
2. kfree(vmx.nested.cached_shadow_vmcs12).
3. vmx.nested.cached_shadow_vmcs12 = NULL.

### Out of Scope

- KVM nested-VMX core (covered in `x86-nested-vmx.md` Tier-3)
- KVM VMCS layout (covered in `x86-vmx.md` Tier-3)
- KVM VMX VMENTER (covered in `x86-vmx-vmenter.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enable_shadow_vmcs` (module-param) | per-host enable | `VmxOps::enable_shadow_vmcs` |
| `struct shadow_vmcs_field` | per-field bitmap entry | `ShadowVmcsField` |
| `shadow_read_only_fields[]` | per-RO field list | `Vmx::shadow_ro_fields` |
| `shadow_read_write_fields[]` | per-RW field list | `Vmx::shadow_rw_fields` |
| `init_vmcs_shadow_fields()` | per-bitmap construction | `Vmx::init_shadow_fields` |
| `vmx->vmcs01.shadow_vmcs` | per-VMCS01 shadow-vmcs page | `LoadedVmcs::shadow_vmcs` |
| `vmx->nested.cached_shadow_vmcs12` | per-shadow-vmcs12 cache | `Nested::cached_shadow_vmcs12` |
| `vmx->nested.shadow_vmcs12_cache` | per-VMCS12 GHC | `Nested::shadow_vmcs12_cache` |
| `vmx_disable_shadow_vmcs()` | per-vmptrld disable | `Vmx::disable_shadow_vmcs` |
| `nested_cache_shadow_vmcs12()` | per-shadow12 cache fill | `Nested::cache_shadow_vmcs12` |
| `copy_shadow_to_vmcs12()` | per-vmexit shadow → vmcs12 | `Nested::copy_shadow_to_vmcs12` |
| `copy_vmcs02_to_shadow()` | per-vmenter copy | `Nested::copy_vmcs02_to_shadow` |
| `VMREAD_BITMAP` (vmcs.field 0x2024) | per-VMCS read-bitmap | UAPI |
| `VMWRITE_BITMAP` (vmcs.field 0x2026) | per-VMCS write-bitmap | UAPI |

### compatibility contract

REQ-1: Shadow-VMCS hardware capability:
- vmcs.SECONDARY_VM_EXEC_CONTROL bit SHADOW_VMCS available.
- enable_shadow_vmcs module-param defaults true if available.

REQ-2: Per-field bitmap (4KB pages):
- VMREAD bitmap (bytes [0..4095]): bit per VMCS field encoded as 16-bit field-encoding.
- VMWRITE bitmap (separate 4KB): symmetric.
- Bit set: vmexit on VMREAD/VMWRITE for that field.
- Bit clear: HW handles directly via shadow-VMCS.

REQ-3: vmcs_shadow_fields.h:
- Lists fields to be shadowed (RO + RW separately).
- RO examples: GUEST_RIP, GUEST_RSP, GUEST_RFLAGS, GUEST_INTERRUPTIBILITY_INFO.
- RW examples: GUEST_CR0, GUEST_CR3, GUEST_DR7, etc.

REQ-4: init_vmcs_shadow_fields:
- Per-RO field: clear VMREAD-bitmap bit; set VMWRITE-bitmap bit (RO not writable).
- Per-RW field: clear both bitmaps.

REQ-5: Per-vCPU vmcs01.shadow_vmcs:
- alloc_vmcs() at vcpu-create.
- vmcs_clear(shadow_vmcs).
- vmcs01.SECONDARY_VM_EXEC_CONTROL |= SHADOW_VMCS.
- vmcs01.VMREAD_BITMAP / VMWRITE_BITMAP point to per-CPU bitmaps.
- vmcs01.VMCS_LINK_POINTER = phys(shadow_vmcs).

REQ-6: Per-VMPTRLD (L1 issues VMPTRLD on vmcs12-pa):
- KVM caches vmcs12 at vcpu.arch.vmptr.
- nested_cache_shadow_vmcs12 reads vmcs12 from guest mem; populates shadow_vmcs.
- VMCS_LINK_POINTER pointed to.

REQ-7: Per-VMRUN entry to L2:
- copy_vmcs02_to_shadow: copy vmcs12 fields to shadow_vmcs.
- HW sees shadow_vmcs at L2 VMREAD/VMWRITE.

REQ-8: Per-VMEXIT from L2:
- copy_shadow_to_vmcs12: HW updates flushed back to vmcs12.

REQ-9: VMCLEAR on shadow_vmcs:
- Per-VMCS context-clear: vmcs_clear(shadow_vmcs).
- Per-VMPTRLD swap.

REQ-10: enable_shadow_vmcs=0 fallback:
- Per-VMREAD/VMWRITE always vmexit.
- KVM emulates.

REQ-11: Per-shadow_vmcs lifetime:
- Allocated per-vCPU at vcpu-create.
- Freed per-vCPU at vcpu-destroy.

REQ-12: Per-cached_shadow_vmcs12:
- Per-vCPU cached vmcs12 (sw-side) for KVM to lookup w/o guest-mem read.
- Refreshed on VMPTRLD.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bitmap_pages_4k` | INVARIANT | per-bitmap 4KB. |
| `read_only_field_vmwrite_intercepted` | INVARIANT | per-RO field: VMWRITE-bitmap bit set. |
| `read_write_field_both_clear` | INVARIANT | per-RW field: both bitmap bits clear. |
| `shadow_vmcs_lifetime_eq_vcpu` | INVARIANT | shadow_vmcs lifetime ≤ vCPU lifetime. |
| `enable_shadow_vmcs_gates_alloc` | INVARIANT | !enable_shadow_vmcs ⟹ shadow_vmcs == NULL. |

### Layer 2: TLA+

`virt/kvm/nested_shadow_vmcs.tla`:
- Per-VMPTRLD shadow-vmcs cache + per-vmenter copy + per-vmexit drain.
- Properties:
  - `safety_no_l2_read_unintercepted_unshadowed` — per-non-shadowed VMREAD ⟹ vmexit.
  - `safety_shadow_consistent_with_vmcs12` — post-vmexit: vmcs12 fields equal post-copy.
  - `liveness_l2_vmread_no_vmexit_for_shadowed` — per-shadowed-field VMREAD ⟹ no vmexit.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vmx::init_shadow_fields` post: bitmaps populated per shadow_*_fields | `Vmx::init_shadow_fields` |
| `Nested::copy_vmcs02_to_shadow` post: shadow_vmcs has vmcs12 field values | `Nested::copy_vmcs02_to_shadow` |
| `Nested::copy_shadow_to_vmcs12` post: vmcs12 has shadow_vmcs field values | `Nested::copy_shadow_to_vmcs12` |
| `Vmx::disable_shadow_vmcs` post: SHADOW_VMCS bit cleared | `Vmx::disable_shadow_vmcs` |

### Layer 4: Verus/Creusot functional

`Per-L2 VMREAD/VMWRITE shadowed-field → HW handles via shadow_vmcs no-vmexit; per-non-shadowed → vmexit + KVM emulate` semantic equivalence: per-Intel SDM §29.7 nested-VMX shadow-VMCS spec.

### hardening

(Inherits row-1 features from `virt/kvm/x86-nested-vmx.md` § Hardening.)

Shadow-VMCS-specific reinforcement:

- **Per-bitmap default-intercept** — defense against per-newly-added field unexpected exposure.
- **Per-RO field VMWRITE intercepted** — defense against L2 mutating RO field.
- **Per-shadow_vmcs alloc per-vCPU** — defense against per-vCPU state leak.
- **Per-VMCS_LINK_POINTER set during shadow active** — defense against per-vmenter null link causing CPU exception.
- **Per-cached_shadow_vmcs12 refreshed on VMPTRLD** — defense against per-stale cache.
- **Per-init_shadow_fields runs once** — defense against per-init race.
- **Per-disable_shadow_vmcs at vcpu-destroy** — defense against per-stale shadow on next-VM-create.
- **Per-shadow_vmcs page locked** — defense against host swap moving page mid-vmenter.
- **Per-bitmap pages per-CPU shared** — defense against per-vCPU bitmap divergence.
- **Per-EPT_POINTER intercepted always** — defense against per-L2 EPT-root spoofing.
- **Per-shadow_read_only/rw_fields curated** — defense against per-field accidentally-shadowed mutation.

