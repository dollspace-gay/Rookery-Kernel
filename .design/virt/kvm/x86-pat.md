# Tier-3: arch/x86/kvm/x86.c (PAT subset) — KVM PAT (Page-Attribute-Table) MSR virtualization

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/x86.c (MSR_IA32_CR_PAT cases at ~4027 + ~4438 + ~13084)
  - arch/x86/kvm/vmx/vmx.c (~2434, ~4463, ~7823 vmx_ignore_guest_pat, ~7844)
  - arch/x86/kvm/svm/svm.c (~2968)
  - arch/x86/include/asm/msr-index.h (MSR_IA32_CR_PAT 0x277)
  - arch/x86/mm/pat/memtype.c (PAT-MSR encoding)
-->

## Summary

Intel/AMD PAT (Page-Attribute-Table; SDM-Vol3 §13.12 / APM-Vol2 §7.7) is an 8-entry MSR (`MSR_IA32_CR_PAT` 0x277) selecting per-PTE memory-type from PWT|PCD|PAT bits. Per-entry encoding: 0=UC, 1=WC, 4=WT, 5=WP, 6=WB, 7=UC-. KVM virtualizes via per-vCPU `vcpu.arch.pat` shadow + per-VMX `vmcs.guest_ia32_pat` auto-load + per-SVM intercept. Critical for: guest MTRR + PCI-MMIO mapping + per-GPU framebuffer. v0 reproduces Linux 7.1.0-rc2 PAT semantics including `vmx_ignore_guest_pat` quirk for non-coherent DMA.

This Tier-3 covers PAT MSR virtualization in `arch/x86/kvm/x86.c` + per-vendor at `vmx.c` and `svm.c` (~80 lines combined).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `MSR_IA32_CR_PAT` (0x277) | PAT control MSR | UAPI |
| `MSR_IA32_CR_PAT_DEFAULT` | x86 reset value | shared |
| `vcpu->arch.pat` | per-vCPU shadow PAT | `Vcpu::arch::pat` |
| `kvm_pat_valid(pat)` | per-byte memtype validation | `Pat::valid` |
| `vmx_ignore_guest_pat(kvm)` | per-VM quirk for ignored guest PAT | `VmxOps::ignore_guest_pat` |
| `vmcs.guest_ia32_pat` | per-vCPU VMX auto-load PAT | `Vmcs::guest_ia32_pat` |
| `VM_ENTRY_LOAD_IA32_PAT` | vmcs.entry-controls bit | shared |
| `VM_EXIT_SAVE_IA32_PAT` | vmcs.exit-controls bit | shared |
| `vmcb.save.g_pat` | per-vCPU SVM g_pat | `Vmcb::g_pat` |
| `kvm_set_msr_common()` | MSR_IA32_CR_PAT WRMSR handler | `Vcpu::handle_msr` |
| `kvm_get_msr_common()` | MSR_IA32_CR_PAT RDMSR handler | `Vcpu::handle_msr` |

## Compatibility contract

REQ-1: `MSR_IA32_CR_PAT` (0x277) layout:
- 8 entries × 8 bits = 64 bits.
- Per-entry value: 0 (UC), 1 (WC), 4 (WT), 5 (WP), 6 (WB), 7 (UC-).
- Per-entry indexed by `(PAT << 2) | (PCD << 1) | PWT` from PTE/PDE.

REQ-2: Reset value:
- `MSR_IA32_CR_PAT_DEFAULT = 0x0007040600070406`.
- Per-vCPU init: `vcpu.arch.pat = MSR_IA32_CR_PAT_DEFAULT`.

REQ-3: WRMSR(MSR_IA32_CR_PAT, value):
- Per-byte (8 bytes) validated: byte_n & 0xF8 == 0 ∧ byte_n & 0x07 ∈ {0,1,4,5,6,7}.
- Reserved values (2, 3) → #GP.
- Per-byte upper-5-bits (bits 3-7 of each byte) zero → #GP if non-zero.

REQ-4: RDMSR(MSR_IA32_CR_PAT):
- Return `vcpu.arch.pat`.

REQ-5: VMX path:
- vmcs.entry-controls VM_ENTRY_LOAD_IA32_PAT = 1 → guest PAT loaded from vmcs.guest_ia32_pat.
- vmcs.exit-controls VM_EXIT_SAVE_IA32_PAT = 1 → host PAT saved on vmexit.
- Per-WRMSR(IA32_CR_PAT) by guest: vmcs.guest_ia32_pat updated.

REQ-6: VMX `vmx_ignore_guest_pat` quirk:
- For VMX without EPT-MT-bypass (or per-VM `KVM_X86_QUIRK_IGNORE_GUEST_PAT`):
- Per-VM-flag set → vcpu.arch.pat shadowed but EPT memtype determined by per-EPT-PTE memtype from `kvm_arch_dirty_log_supported()` + memslot flags.
- Per-VM with non-coherent DMA: must NOT ignore-guest-pat.

REQ-7: SVM path:
- `vmcb.save.g_pat` directly mapped to guest PAT.
- WRMSR(IA32_CR_PAT) updates vmcb.save.g_pat; RDMSR returns vmcb.save.g_pat.
- vmcb.intercepts: MSR_IA32_CR_PAT typically NOT intercepted (passthrough via MSR-bitmap).

REQ-8: Per-EPT/NPT PTE memtype:
- Per-EPT-PTE bits[5:3]: ept-memtype (0=UC, 1=WC, 4=WT, 5=WP, 6=WB).
- For DMA-coherent VMs: ept-memtype = WB.
- For DMA-non-coherent VMs: ept-memtype = guest-PAT-resolved.

REQ-9: Per-vCPU CPUID gating:
- CPUID 0x01:EDX[16] PAT (always available on x86_64 v0).
- Guest may always WRMSR/RDMSR PAT (no gating).

REQ-10: Live migration:
- Per-vCPU vcpu.arch.pat migrated via KVM_GET_MSRS / KVM_SET_MSRS.
- Destination per-VMX/SVM vmcs/vmcb resynced.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest: `cat /proc/cpuinfo` shows `pat` flag.
- [ ] AC-2: Per-vCPU init: RDMSR(IA32_CR_PAT) = 0x0007040600070406.
- [ ] AC-3: Guest WRMSR(IA32_CR_PAT, 0x0007040600070406): RDMSR returns same.
- [ ] AC-4: Reserved-byte WRMSR (e.g. 0x0007040600070403): #GP.
- [ ] AC-5: VMX guest WRMSR: vmcs.guest_ia32_pat updated.
- [ ] AC-6: SVM guest WRMSR: vmcb.save.g_pat updated.
- [ ] AC-7: Per-VM KVM_X86_QUIRK_IGNORE_GUEST_PAT: ept-memtype = WB regardless of guest PAT.
- [ ] AC-8: Per-VM with non-coherent assigned device: ignore-guest-pat = false; ept-memtype = guest PAT-resolved.
- [ ] AC-9: Live migration: per-vCPU PAT preserved.
- [ ] AC-10: kvm-unit-tests `pat` test passes.

## Architecture

Per-vCPU PAT shadow:

```
struct VcpuArch {
  ...
  pat: u64,                                      // shadow of MSR_IA32_CR_PAT
}
```

Per-VM ignore-quirk:

```
struct KvmArch {
  ...
  disabled_quirks: u64,                           // KVM_X86_QUIRK_*
  noncoherent_dma_count: AtomicU32,               // > 0 ⟹ no-quirk
}
```

`Pat::valid(pat)`:
1. For each of 8 bytes:
   - byte = (pat >> (i*8)) & 0xFF
   - if byte & 0xF8 != 0: return false.
   - if byte & 0x07 ∈ {2, 3}: return false.
2. return true.

`Vcpu::handle_msr` MSR_IA32_CR_PAT branch (write):
1. If !Pat::valid(data): #GP.
2. vcpu.arch.pat = data.
3. If vmx-active: vmcs.guest_ia32_pat = data.
4. If svm-active: vmcb.save.g_pat = data.

`Vcpu::handle_msr` MSR_IA32_CR_PAT branch (read):
1. Return vcpu.arch.pat.

`VmxOps::ignore_guest_pat(kvm)`:
1. If kvm.arch.noncoherent_dma_count > 0: return false.
2. If kvm.arch.disabled_quirks & KVM_X86_QUIRK_IGNORE_GUEST_PAT: return false.
3. Return true (default for VMX without IPAT).

`Mmu::ept_memtype(gfn, vcpu)`:
1. If VmxOps::ignore_guest_pat(kvm): return WB.
2. Compute per-PTE PAT entry index from guest PTE bits:
   - pat_idx = ((pte >> 7) & 1) << 2 | ((pte >> 4) & 1) << 1 | ((pte >> 3) & 1).
   - memtype = (vcpu.arch.pat >> (pat_idx * 8)) & 0x07.
3. Return memtype.

`SvmOps::npt_memtype(gfn, vcpu)`:
1. SVM NPT uses guest PAT directly via vmcb.save.g_pat.
2. Per-PTE memtype = guest-PAT-resolved.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pat_valid_iff_per_byte_valid` | INVARIANT | Pat::valid(p) ⟺ all 8 bytes have memtype ∈ {0,1,4,5,6,7} ∧ upper-5-bits zero. |
| `pat_default_value_valid` | INVARIANT | Pat::valid(MSR_IA32_CR_PAT_DEFAULT). |
| `vmcs_guest_pat_eq_arch_pat` | INVARIANT | vmcs.guest_ia32_pat == vcpu.arch.pat after WRMSR. |
| `vmcb_g_pat_eq_arch_pat` | INVARIANT | vmcb.save.g_pat == vcpu.arch.pat after WRMSR. |

### Layer 2: TLA+

`virt/kvm/pat.tla`:
- Per-vCPU PAT WRMSR/RDMSR sequence + EPT/NPT memtype.
- Properties:
  - `safety_pat_valid_after_wrmsr` — per-WRMSR success ⟹ post.pat satisfies Pat::valid.
  - `safety_gp_on_invalid_pat` — per-WRMSR with reserved byte ⟹ #GP injected; pat unchanged.
  - `safety_ept_memtype_resolves` — per-EPT memtype is well-defined for any guest PTE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::handle_msr` PAT-write post: vcpu.arch.pat == data; vmcs.guest_ia32_pat == data (vmx) | `Vcpu::handle_msr` |
| `Vcpu::handle_msr` PAT-read post: returned-value == vcpu.arch.pat | `Vcpu::handle_msr` |
| `Mmu::ept_memtype` post: returned ∈ {UC, WC, WT, WP, WB, UC-} | `Mmu::ept_memtype` |
| `VmxOps::ignore_guest_pat` post: false ⟹ noncoherent_dma_count > 0 ∨ quirk-disabled | `VmxOps::ignore_guest_pat` |

### Layer 4: Verus/Creusot functional

`Per-WRMSR(IA32_CR_PAT, v) → guest sees v on RDMSR; per-EPT-PTE access uses PAT-resolved memtype` semantic equivalence: per-instruction PAT semantics match Intel SDM §13.12 + AMD APM §7.7.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

PAT-specific reinforcement:

- **Per-byte validation** — defense against guest writing reserved memtype causing CPU exception.
- **Per-VM noncoherent_dma_count gates ignore-quirk** — defense against assigned-device DMA seeing wrong-memtype.
- **Per-vCPU PAT shadowed in vmcs/vmcb** — defense against per-vCPU stale memtype on vmenter.
- **MSR_IA32_CR_PAT passthrough only after CPUID-PAT advertised** — defense against guest accessing nonexistent MSR.
- **Per-PTE EPT memtype resolved deterministically** — defense against memtype-aliasing attack on cache.
- **Default reset value enforced at vCPU init** — defense against guest seeing host PAT value.
- **Live-migrate per-vCPU PAT** — defense against post-migrate cache-coherency mismatch.
- **Per-VM KVM_X86_QUIRK_IGNORE_GUEST_PAT requires explicit opt-out** — defense against silent EPT-WB override.
- **PAT consistency check across vCPUs in same VM** — defense against per-vCPU disagreement on memtype causing TLB-broadcast issues.
- **Audit per-WRMSR(IA32_CR_PAT) on policy-restricted VMs** — defense against policy bypass via memtype change.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — KVM_SET_MSRS IA32_CR_PAT payload bounded.
- **PAX_KERNEXEC** — PAT MSR write-handler + memtype resolver RO after init.
- **PAX_RANDKSTACK** — randomized kstack per WRMSR(PAT) emulation.
- **PAX_REFCOUNT** — noncoherent_dma_count saturating.
- **PAX_MEMORY_SANITIZE** — per-vCPU PAT cache zeroed on destroy.
- **PAX_UDEREF** — KVM_SET_MSRS user pointer validated.
- **PAX_RAP / kCFI** — pat-update / memtype-resolve callbacks type-checked.
- **GRKERNSEC_HIDESYM** — PAT raw value redacted in trace.
- **GRKERNSEC_DMESG** — PAT-quirk warnings rate-limited.

PAT-specific:

- **CAP_SYS_ADMIN strict on KVM_SET_MSRS(IA32_CR_PAT)** — defense against unprivileged memtype flip.
- **Reserved per-byte memtype rejected at WRMSR** — closes CPU-exception surface.
- **KVM_X86_QUIRK_IGNORE_GUEST_PAT explicit opt-out** — defense against silent EPT-WB override.

Rationale: PAT controls per-page memtype seen by both guest CPU and assigned DMA devices; sanitize + CAP-gate close the cross-VM memtype-aliasing surface that would enable cache-side-channels.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- KVM EPT MMU (covered in `kvm-x86-mmu-ept.md` Tier-3)
- KVM NPT MMU (covered in `kvm-x86-mmu-npt.md` Tier-3)
- KVM CR0/CR4 (covered in `x86-cr0-cr4.md` Tier-3)
- Per-arch host PAT init (arch/x86/mm/pat/memtype.c; covered separately)
- KVM MTRR (covered in `x86-mtrr.md` future Tier-3)
- Implementation code
