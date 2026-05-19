# Tier-3: arch/x86/kvm/vmx/vmx.c (segment subset) — segment-register virtualization (CS/DS/ES/SS/FS/GS/LDTR/TR + descriptor-cache)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-vmx.md
upstream-paths:
  - arch/x86/kvm/vmx/vmx.c (segment sections; see grep "vmx_get_segment|vmx_set_segment|VCPU_SREG_")
  - arch/x86/kvm/svm/svm.c (svm_get_segment / _set_segment)
  - arch/x86/kvm/x86.c (kvm_arch_get_segment + segment-register migration)
-->

## Summary

x86 segment registers (CS, DS, ES, SS, FS, GS, LDTR, TR) carry per-segment descriptor-cache (base, limit, access-rights) loaded from GDT/LDT on segment-load. KVM virtualizes these per-vCPU: vmcs.GUEST_CS_BASE / _LIMIT / _AR_BYTES per-segment fields directly observable; vmcb.save.cs/ds/es/ss/fs/gs/tr/ldtr similarly. Per-segment AR (Access Rights) byte encodes type, S, DPL, P, AVL, L, D/B, G. Per-mode validation: real-mode CS = SS = DS = ES = FS = GS aligned; long-mode CS.L=1; legacy 32-bit CS.D=1. KVM also maintains `vcpu.arch.segment_cache` to avoid repeated VMREAD on hot paths.

This Tier-3 covers segment-register virtualization in `vmx.c` (~300-400 lines) + `svm.c` analog + `x86.c` arch-agnostic.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enum kvm_reg` (VCPU_SREG_*) | per-segment ID | `KvmSreg` |
| `kvm_segment` struct | segment descriptor (base, limit, AR, etc.) | `KvmSegment` |
| `vmx_get_segment(vcpu, var, seg)` | VMX read segment from vmcs | `VcpuVmx::get_segment` |
| `vmx_set_segment(vcpu, var, seg)` | VMX write segment to vmcs | `VcpuVmx::set_segment` |
| `svm_get_segment(vcpu, var, seg)` (svm.c) | SVM read | `VcpuSvm::get_segment` |
| `svm_set_segment(vcpu, var, seg)` (svm.c) | SVM write | `VcpuSvm::set_segment` |
| `kvm_arch_get_segment(vcpu, &reg, seg)` | per-arch dispatch | `Vcpu::get_segment` |
| `kvm_arch_set_segment(vcpu, &reg, seg)` | per-arch dispatch | `Vcpu::set_segment` |
| `vmx_segment_cache_*` | per-vCPU segment-read cache | `VcpuVmx::segment_cache` |
| `kvm_make_segments_consistent(vcpu)` | per-mode consistency-fix | `Vcpu::make_segments_consistent` |
| `vmx_set_segment_real_mode(...)` | real-mode segment fix-up | `VcpuVmx::set_segment_real_mode` |
| `enter_rmode(vcpu)` | switch to real-mode emulation | `VcpuVmx::enter_rmode` |
| `enter_pmode(vcpu)` | switch to protected-mode | `VcpuVmx::enter_pmode` |
| `seg_setup(seg)` | per-segment default setup | `VcpuVmx::seg_setup` |
| `vmx_get_cpl(vcpu)` | per-vCPU current privilege level | `VcpuVmx::get_cpl` |

## Compatibility contract

REQ-1: `kvm_segment` struct (per-segment descriptor):
- `base` (u64; per-segment base addr).
- `limit` (u32; per-segment limit).
- `selector` (u16; segment selector).
- `type_` (4 bits; code/data/system type).
- `present` (1 bit; P).
- `dpl` (2 bits; DPL).
- `db` (1 bit; D/B; 32-bit code/stack vs 16-bit).
- `s` (1 bit; system-segment if 0).
- `l` (1 bit; long-mode for CS).
- `g` (1 bit; granularity).
- `avl` (1 bit; available).
- `unusable` (1 bit; segment unusable).

REQ-2: VCPU_SREG_* IDs:
- VCPU_SREG_ES (0), VCPU_SREG_CS (1), VCPU_SREG_SS (2), VCPU_SREG_DS (3).
- VCPU_SREG_FS (4), VCPU_SREG_GS (5), VCPU_SREG_TR (6), VCPU_SREG_LDTR (7).

REQ-3: VMX vmcs segment fields:
- GUEST_CS_SELECTOR, GUEST_CS_BASE, GUEST_CS_LIMIT, GUEST_CS_AR_BYTES.
- Same for ES / SS / DS / FS / GS / TR / LDTR.
- AR_BYTES encoding:
  - Bits[3:0] type.
  - Bit[4] S.
  - Bits[6:5] DPL.
  - Bit[7] P.
  - Bit[12] AVL.
  - Bit[13] L.
  - Bit[14] D/B.
  - Bit[15] G.
  - Bit[16] unusable.

REQ-4: SVM vmcb segment fields:
- vmcb.save.cs / ds / es / ss / fs / gs / ldtr / tr.
- Each: { selector, attrib, limit, base }.
- attrib encodes packed type/S/DPL/P/AVL/L/D/G.

REQ-5: Per-vCPU `segment_cache`:
- Per-segment cache per-VMREAD result.
- Flag bits to invalidate cache on VMREAD/segment-write.
- Reduces VMREAD overhead on hot paths.

REQ-6: VMX get_segment flow:
1. Read VMCS GUEST_<SEG>_SELECTOR / _BASE / _LIMIT / _AR_BYTES.
2. Decode AR_BYTES → type / S / DPL / P / etc.
3. If unusable bit: var.unusable = 1.
4. Cache result.

REQ-7: VMX set_segment flow:
1. Compose AR_BYTES from var.type / var.s / var.dpl / etc.
2. If var.unusable: AR_BYTES |= UNUSABLE_BIT.
3. VMWRITE GUEST_<SEG>_*.
4. Invalidate cache.

REQ-8: Real-mode emulation (per Intel-VMX limitation):
- Real-mode segments must have specific format (base = selector << 4; limit = 0xFFFF; AR = 0x93 for data / 0x9B for code).
- KVM uses shadow `rmode.segs[]` to track real-mode-style segments while running in vm86-emulated mode.
- enter_rmode / enter_pmode swaps state.

REQ-9: Long-mode segment ignore:
- CS.L=1: 64-bit mode; FS/GS bases significant; ES/DS/SS effectively ignored.
- KVM sets ES/DS/SS to flat 64-bit defaults.

REQ-10: TR (Task Register):
- Points to TSS.
- Per-vCPU TR.base used by HW for stack-switch on privilege-level-change (interrupts).
- Must always be valid + present in protected-mode.

REQ-11: LDTR (Local Descriptor Table Register):
- Points to LDT.
- Optional; selector 0 = unused.

REQ-12: Per-segment CPL:
- CPL := CS.DPL (typically) OR SS.DPL (per Intel rules).
- vmx_get_cpl returns per-vCPU current CPL.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest: BIOS sets CS=0xF000, DS=0xF000 in real-mode; transition to pmode with proper GDT.
- [ ] AC-2: Long-mode entry: CS.L=1; subsequent fetches in 64-bit mode.
- [ ] AC-3: Per-CPL: kernel CS.DPL=0; user CS.DPL=3; vmx_get_cpl returns correctly.
- [ ] AC-4: Real-mode emulation: 16-bit DOS guest works via vm86-emulation; segments tracked.
- [ ] AC-5: Multi-CPU SMP: per-vCPU per-CPU FS/GS bases distinct; gsbase/fsbase preserved.
- [ ] AC-6: Live migration: per-vCPU per-segment state migrated; post-migrate guest sees consistent.
- [ ] AC-7: TR programming: guest LTR loads TR; subsequent IRQ uses TSS.SP0 for stack-switch.
- [ ] AC-8: LDTR set: LLDT loads LDTR; subsequent segment-load can use LDT entries.
- [ ] AC-9: Segment-cache: hot-path VMREAD avoided for unchanged segments; counter shows reduction.
- [ ] AC-10: kvm-unit-tests `segment` test passes.

## Architecture

`KvmSegment`:

```
struct KvmSegment {
  base: u64,
  limit: u32,
  selector: u16,
  type_: u8,           // 4 bits
  present: u8,         // 1 bit
  dpl: u8,             // 2 bits
  db: u8,              // 1 bit
  s: u8,               // 1 bit
  l: u8,               // 1 bit
  g: u8,               // 1 bit
  avl: u8,             // 1 bit
  unusable: u8,        // 1 bit
  padding: u8,
}
```

`VcpuVmx::get_segment(vcpu, &var, seg)`:
1. var.selector = vmcs_read16(GUEST_<SEG>_SELECTOR).
2. var.base = vmcs_readl(GUEST_<SEG>_BASE).
3. var.limit = vmcs_read32(GUEST_<SEG>_LIMIT).
4. ar := vmcs_read32(GUEST_<SEG>_AR_BYTES).
5. Decode ar: var.type = ar & 0xF; var.s = (ar >> 4) & 1; var.dpl = (ar >> 5) & 0x3; var.present = (ar >> 7) & 1; var.avl = (ar >> 12) & 1; var.l = (ar >> 13) & 1; var.db = (ar >> 14) & 1; var.g = (ar >> 15) & 1; var.unusable = (ar >> 16) & 1.

`VcpuVmx::set_segment(vcpu, &var, seg)`:
1. ar := var.type | (var.s << 4) | (var.dpl << 5) | (var.present << 7) | (var.avl << 12) | (var.l << 13) | (var.db << 14) | (var.g << 15) | (var.unusable << 16).
2. vmcs_write16(GUEST_<SEG>_SELECTOR, var.selector).
3. vmcs_writel(GUEST_<SEG>_BASE, var.base).
4. vmcs_write32(GUEST_<SEG>_LIMIT, var.limit).
5. vmcs_write32(GUEST_<SEG>_AR_BYTES, ar).
6. Invalidate per-vCPU segment_cache for seg.

`VcpuVmx::enter_rmode(vcpu)`:
1. Snapshot per-segment to rmode.segs[].
2. Set CS / DS / ES / SS / FS / GS to real-mode-compatible:
   - selector = base >> 4 (or original >> 4).
   - base = selector << 4.
   - limit = 0xFFFF.
   - AR_BYTES = 0x9B (CS) / 0x93 (data).
3. Set CR0.PE = 0.
4. vcpu.arch.real_mode_entry = true.

`VcpuVmx::enter_pmode(vcpu)`:
1. Restore per-segment from rmode.segs[].
2. CR0.PE = 1.
3. vcpu.arch.real_mode_entry = false.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `seg_idx_bounded` | OOB | per-segment idx ∈ [0, 7]; defense against OOB segment-cache access. |
| `selector_consistent_with_base_in_real_mode` | INVARIANT | real-mode: base == selector << 4. |
| `cs_l_consistent_with_long_mode` | INVARIANT | long-mode: CS.L = 1; CS.D = 0. |
| `ar_bytes_packed_correctly` | INVARIANT | encode/decode round-trip preserves values. |
| `unusable_bit_excludes_use` | INVARIANT | unusable segment cannot resolve memory access. |

### Layer 2: TLA+

`virt/kvm/segment_state_machine.tla`:
- Per-segment state ∈ {Unused, Real, Pmode, LongMode}.
- Properties:
  - `safety_real_mode_format` — Real implies base == selector << 4.
  - `safety_long_mode_cs_L` — LongMode implies CS.L == 1.
  - `safety_no_LongMode_with_compat_segment` — LongMode implies CS.L == 1, ES/DS/SS unrestricted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuVmx::get_segment` post: var fields populated from vmcs reads | `VcpuVmx::get_segment` |
| `VcpuVmx::set_segment` post: vmcs fields written; cache invalidated | `VcpuVmx::set_segment` |
| `VcpuVmx::enter_rmode` post: rmode.segs[] populated; CS/DS/etc real-mode formatted | `VcpuVmx::enter_rmode` |
| Per-vCPU CPL == CS.DPL (or SS.DPL) per Intel SDM rules | `VcpuVmx::get_cpl` |

### Layer 4: Verus/Creusot functional

`Per-segment-write: subsequent vmenter loads segment with these values; subsequent guest CR/seg-read returns consistent state` semantic equivalence: per-segment round-trip preserves all encoded fields.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

segment-virtualization-specific reinforcement:

- **Per-segment AR_BYTES validated** — defense against malformed bits causing CPU-#GP at vmenter.
- **Real-mode format enforced** — defense against vmenter consistency-check failure.
- **Long-mode CS.L=1 + CS.D=0 strict** — defense against legacy-32-bit-execution-attempt in long-mode.
- **TR.present always 1 in protected-mode** — defense against IRQ-during-no-TSS causing #DF.
- **Per-vCPU segment_cache invalidation** at relevant transitions — defense against stale-cache-read.
- **Per-vCPU CPL determination atomic** — defense against torn CPL reading wrong DPL.
- **rmode shadow segs separated from pmode** — defense against mode-mixing causing inconsistent state.
- **Live-migrate per-segment validated against destination CPUID** — defense against unsupported-feature.
- **Per-vmenter consistency-check** — defense against guest entering with malformed segment state.
- **TR / LDTR descriptor-table-pointers validated** — defense against pointing to invalid GDT entry.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — KVM_GET_SREGS / KVM_SET_SREGS copies bounded by `struct kvm_sregs` slab size; per-segment fields validated before VMCS/VMCB load.
- **PAX_KERNEXEC** — vendor `get_segment` / `set_segment` / `get_cpl` ops resolve through RO `kvm_x86_ops` vtable; module text RX-only.
- **PAX_RANDKSTACK** — VM-entry / segment-cache invalidation paths inherit RANDKSTACK from KVM_RUN.
- **PAX_REFCOUNT** — per-vCPU segment-cache (`segment_cache`) and per-vCPU CPL state saturating refcount through vCPU lifetime.
- **PAX_MEMORY_SANITIZE** — `vcpu.arch.segment_cache`, `regs_avail`, `regs_dirty` zeroed on vCPU alloc/free so stale segment-base/limit cannot leak to a successor vCPU.
- **PAX_UDEREF** — KVM_SET_SREGS copy_from_user STAC/CLAC bracketed; no raw user pointer follows the segment-write path.
- **PAX_RAP / kCFI** — `kvm_x86_ops.get_segment`, `set_segment`, `get_cpl`, `set_idt`, `set_gdt` slots RAP-signed.
- **GRKERNSEC_HIDESYM** — per-vCPU segment-cache kaddr and CS/SS/TR base redacted unless CAP_SYSLOG + gr-rbac.
- **GRKERNSEC_DMESG** — invalid-segment / VMCS-consistency-check warnings rate-limited and gated by `dmesg_restrict`.
- **KVM ioctl CAP_SYS_ADMIN strict** — KVM_SET_SREGS / KVM_SET_SREGS2 gated to CAP_SYS_ADMIN + gr-rbac VM-create role.
- **VMCS / VMCB segment field validation** — per-segment selector/base/limit/AR consistency enforced before VMRESUME; malformed state rejected at KVM_SET_SREGS rather than at vmentry.
- **CPL ↔ CS.RPL atomic read** — per-vCPU CPL computed atomically from CS.RPL so torn-read cannot expose stale ring-level to permission checks.
- **TR / LDTR descriptor pointer validation** — TR/LDTR base validated against GDT bounds; pointing to invalid descriptor rejected.
- **Nested-virt strict** — L2 segment state validated independently of L1's VMCS shadow; L0 host segment state never directly written by L2.

Per-doc rationale: segments are the foundation of x86 ring-level enforcement; the grsec reinforcement here validates segment-cache writes early (at KVM_SET_SREGS) rather than at vmentry, atomic-reads CPL to defeat torn-read CPL race, and SANITIZEs the per-vCPU segment cache so stale CS/SS/TR base addresses cannot bleed across vCPU lifetimes.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VMX core (covered in `x86-vmx.md` Tier-3)
- SVM core (covered in `x86-svm.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- CR0/CR4 (covered in `x86-cr0-cr4.md` Tier-3)
- GDT/LDT/IDT page-walk (covered in `x86-mmu-tdp.md` / `x86-mmu-shadow.md` Tier-3s)
- Implementation code
