---
title: "Tier-3: arch/x86/kvm/x86.c (FPU subset) — KVM FPU/XSAVE virtualization (per-vCPU guest_fpu + XCR0 + XFD)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

KVM virtualizes the x86 FPU/SSE/AVX/AVX-512/AMX context for each guest vCPU via per-vCPU `guest_fpu` (struct fpu) — a self-contained FPU state buffer separate from host kernel's per-task FPU. Per-vmenter: `kvm_load_guest_fpu` swaps host FPU state out + guest FPU state in via XSAVE/XRSTOR (or per-component XSAVES if supported). Per-vmexit: `kvm_put_guest_fpu` reverses. Per-vCPU XCR0 (eXtended Control Register 0) shadow controls which XSAVE components active; XFD (eXtended Feature Disable) MSR allows per-feature lazy-allocation (AMX uses 8KB extra; not allocated unless guest enables). Critical for: full-FPU-state save/restore on every vmenter/vmexit; without it guest would corrupt host FPU + vice-versa.

This Tier-3 covers FPU/XSAVE virtualization in `arch/x86/kvm/x86.c` (~300-400 lines).

### Acceptance Criteria

- [ ] AC-1: Boot Linux guest: vcpu.arch.guest_fpu allocated; xcr0 = 1 (x87 only).
- [ ] AC-2: Guest XSETBV: enable AVX (XCR0 bit 2); subsequent vmenter loads AVX state.
- [ ] AC-3: AMX lazy-alloc: guest XSETBV with AMX bits; XFD-trap fires; KVM realloc fpstate; AMX usable.
- [ ] AC-4: Host FPU isolation: host kernel uses FPU concurrently with guest; no corruption.
- [ ] AC-5: KVM_GET_XSAVE2 + KVM_SET_XSAVE roundtrip: XSAVE-area preserved across migration.
- [ ] AC-6: KVM_GET_XCRS + _SET_XCRS roundtrip: XCR0/XCR1 preserved.
- [ ] AC-7: SEV-ES guest: fpstate confidential; KVM_GET_XSAVE returns -EINVAL.
- [ ] AC-8: AVX-512 perf: guest AVX-512 stress; throughput within 95% of bare-metal.
- [ ] AC-9: Multi-vCPU: 16-vCPU AVX-512 stress; no cross-vCPU FPU state leak.
- [ ] AC-10: kvm-unit-tests `xsave` test passes.

### Architecture

`Vcpu::load_guest_fpu(vcpu)`:
1. KVM_BUG_ON(!vcpu.arch.guest_fpu.fpstate->in_use, vcpu.kvm).
2. fpu_swap_kvm_fpstate(host_fpu, guest_fpu, true).
3. WRMSR(MSR_IA32_XCR0, vcpu.arch.xcr0).
4. WRMSR(MSR_IA32_XFD, vcpu.arch.guest_fpu.fpstate->xfd).
5. trace_kvm_fpu(vcpu_id, 1).

`Vcpu::put_guest_fpu(vcpu)`:
1. fpu_swap_kvm_fpstate(guest_fpu, host_fpu, false).
2. Restore host XCR0 + XFD.
3. trace_kvm_fpu(vcpu_id, 0).

`Vcpu::set_xcr0(vcpu, xcr0)` (XSETBV emulation):
1. Validate: xcr0 ⊆ vcpu.arch.guest_supported_xfeatures.
2. Validate: bit-0 (x87) set.
3. Validate: AVX requires SSE; AVX-512 requires AVX; etc.
4. If new xcr0 includes features not in current fpstate:
   - fpstate_realloc(&vcpu.arch.guest_fpu, new_xcr0).
5. vcpu.arch.xcr0 = xcr0.

`Fpu::update_guest_xfd(guest_fpu, data)`:
1. Validate: data ⊆ XFEATURE_MASK_USER_DYNAMIC.
2. guest_fpu.fpstate.xfd = data.
3. WRMSR(MSR_IA32_XFD, data) (if currently loaded).

`Fpu::fpstate_realloc(guest_fpu, new_xfeatures)`:
1. new_size := required_size(new_xfeatures).
2. If new_size > current_size:
   - Allocate new fpstate; vmalloc if > kmalloc cap.
   - Init from old fpstate.
3. Update guest_fpu.fpstate.

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- SVM vendor (covered in `x86-svm.md` Tier-3)
- SEV-ES VMSA (covered in `x86-sev.md` Tier-3)
- TDX TDVPS (covered in `x86-tdx.md` Tier-3)
- Host kernel FPU API (arch/x86/kernel/fpu; covered in `arch/x86/fpu.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_load_guest_fpu(vcpu)` | per-vmenter swap host→guest FPU | `Vcpu::load_guest_fpu` |
| `kvm_put_guest_fpu(vcpu)` | per-vmexit swap guest→host FPU | `Vcpu::put_guest_fpu` |
| `vcpu->arch.guest_fpu` | per-vCPU guest FPU state | `VcpuArch::guest_fpu` |
| `fpu_swap_kvm_fpstate(...)` (arch/x86/kernel/fpu/core.c) | low-level swap | `Fpu::swap_kvm_fpstate` |
| `kvm_set_msr_xcr0(vcpu, xcr0)` | XCR0 write (covers XSAVE features) | `Vcpu::set_xcr0` |
| `kvm_get_msr_xcr(vcpu, idx, &val)` | XCR read | `Vcpu::get_xcr` |
| `kvm_emulate_xsetbv(vcpu)` | XSETBV instruction emulation | `Vcpu::emulate_xsetbv` |
| `fpu_update_guest_xfd(guest_fpu, data)` | XFD MSR write | `Fpu::update_guest_xfd` |
| `fpstate_realloc(...)` | per-component realloc on XCR0 enable | `Fpu::fpstate_realloc` |
| `kvm_xstate_required_size(xfeatures, compacted)` | per-feature-mask size compute | `Vcpu::xstate_required_size` |
| `kvm_arch_vcpu_ioctl_get_xsave2(vcpu, buf, size)` | KVM_GET_XSAVE2 ioctl | `Vcpu::ioctl_get_xsave2` |
| `kvm_arch_vcpu_ioctl_set_xsave(vcpu, &xsave)` | KVM_SET_XSAVE ioctl | `Vcpu::ioctl_set_xsave` |
| `MSR_IA32_XSS` (0xDA0) | per-vCPU XSAVES supervisor-state mask | UAPI |
| `MSR_IA32_XFD` (0x1C4) | XFD (eXtended Feature Disable) | UAPI |
| `MSR_IA32_XFD_ERR` (0x1C5) | XFD error reporting | UAPI |

### compatibility contract

REQ-1: Per-vCPU `guest_fpu` (struct fpu):
- `fpstate` (KArc<FpState>; XSAVE-area).
- `last_cpu` (per-CPU last-loaded; for cache validation).
- `avx512_timestamp` (AVX-512 use tracking).

REQ-2: Per-fpstate `fpstate`:
- `regs` (XSAVE-area; size depends on XCR0 features; up to ~10KB with AMX).
- `xfeatures` (currently-enabled features bitmap).
- `xrstor_size` (size needed for XRSTOR).
- `is_valloc` (allocated via vmalloc due to size > kmalloc cap).
- `is_confidential` (SEV-ES/TDX guest; KVM cannot read).
- `compacted` (XSAVES compacted format).
- `is_use` (in-use guard).
- `user_xfeatures` (CPUID-allowed user-mode features).

REQ-3: Per-vCPU XCR0:
- vcpu.arch.xcr0 (current XCR0 shadow).
- Per-XSETBV instruction emulation: validate XCR0 features ⊆ guest_supported_xfeatures; update.
- vmcs/vmcb hardware support enables guest XSETBV vmexit-on-write.

REQ-4: Per-vmenter `kvm_load_guest_fpu`:
1. fpu_swap_kvm_fpstate(host → guest):
   - XSAVE host FPU to host_fpu->fpstate.
   - XRSTOR guest_fpu->fpstate.
2. Update XCR0 to guest's XCR0.
3. Update XFD MSR to guest's XFD.
4. Mark fpstate->in_use.

REQ-5: Per-vmexit `kvm_put_guest_fpu`:
1. fpu_swap_kvm_fpstate(guest → host):
   - XSAVE guest FPU to guest_fpu->fpstate.
   - XRSTOR host FPU from host_fpu->fpstate.
2. Restore host XCR0.
3. Restore host XFD.

REQ-6: XCR0 features:
- Bit[0]: x87 (always set).
- Bit[1]: SSE.
- Bit[2]: AVX.
- Bit[3-4]: MPX (deprecated).
- Bit[5-7]: AVX-512 (OPMASK / ZMM_Hi256 / Hi16_ZMM).
- Bit[9]: PT (Intel Processor Trace).
- Bits[17-18]: AMX (TILE-CONFIG / TILE-DATA, large 8KB).
- Bit[19]: APX.

REQ-7: XFD (eXtended Feature Disable):
- MSR for per-feature lazy-allocation.
- When XFD bit set + feature accessed: #NM (Device Not Available) raised.
- KVM intercepts; reallocates fpstate to include feature; clears XFD bit; re-issues instruction.

REQ-8: AMX support gating:
- AMX needs 8KB additional fpstate.
- Per-process limit via /proc/.../arch_status.
- Per-vCPU lazy-alloc on first guest XSETBV including AMX.

REQ-9: KVM_GET_XSAVE / _GET_XSAVE2 / _SET_XSAVE ioctls:
- Per-vCPU read/write entire XSAVE-area.
- Used for live-migration.

REQ-10: KVM_GET_XCRS / _SET_XCRS ioctls:
- Per-vCPU read/write XCR registers (XCR0, XCR1).
- Used for live-migration.

REQ-11: SEV-ES + TDX confidential FPU:
- Per-vCPU fpstate marked confidential.
- KVM cannot read directly; HW manages via VMSA (SEV-ES) or TDVPS (TDX).

REQ-12: per_cpu_ptr(kvm_fpu_swap) optimization:
- Avoid repeated XSAVE/XRSTOR on hot vmenter/vmexit cycles.
- Lazy-write only changed XSAVE components (per-component XSAVES if compacted).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `xcr0_features_valid` | INVARIANT | xcr0 ⊆ guest_supported_xfeatures; required-bit deps enforced. |
| `fpstate_size_matches_xfeatures` | INVARIANT | fpstate.size ≥ required_size(xfeatures). |
| `host_fpu_isolated_from_guest` | INVARIANT | per-vCPU swap restores host fpstate before host kernel uses FPU. |
| `xfd_lazy_alloc_correct` | INVARIANT | XFD-trap → realloc → clear XFD-bit → resume. |
| `confidential_fpu_no_read` | INVARIANT | SEV-ES/TDX fpstate marked confidential; ioctl returns -EINVAL on read. |

### Layer 2: TLA+

`virt/kvm/fpu_swap.tla`:
- Per-vCPU FPU state ∈ {HostInUse, GuestInUse, Saving, Loading}.
- Properties:
  - `safety_no_mid_swap_use` — host kernel uses FPU only when HostInUse.
  - `safety_swap_atomic` — swap intermediate state not visible.
  - `liveness_load_eventually_completes` — Loading → GuestInUse on vmenter complete.

`virt/kvm/xfd_lazy_alloc.tla`:
- Per-vCPU per-feature state: {Disabled, Lazy, Allocated}.
- Properties:
  - `safety_xfd_trap_only_when_lazy` — #NM only fires for Lazy features.
  - `safety_post_alloc_xfd_clear` — post-realloc XFD-bit cleared.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::load_guest_fpu` post: guest fpstate active in HW; host fpstate saved | `Vcpu::load_guest_fpu` |
| `Vcpu::put_guest_fpu` post: host fpstate restored; guest saved to fpstate | `Vcpu::put_guest_fpu` |
| `Vcpu::set_xcr0` post: xcr0 validated; fpstate reallocated if needed | `Vcpu::set_xcr0` |
| `Fpu::update_guest_xfd` post: XFD MSR written; per-CPU state consistent | `Fpu::update_guest_xfd` |

### Layer 4: Verus/Creusot functional

`Per-vCPU FPU state visible to guest matches what guest wrote via XSAVE/XRSTOR; host FPU state preserved across vmenter/vmexit` semantic equivalence: per-cycle full FPU round-trip preserved.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

FPU-virtualization-specific reinforcement:

- **Per-vCPU guest_fpu separate from host fpu** — defense against guest FPU corruption host.
- **XCR0 features validated** — defense against guest enabling unsupported feature causing host #GP.
- **XFD lazy-alloc bounded** — defense against guest enabling AMX without server-side limit.
- **fpstate_realloc atomic w.r.t. vmenter** — defense against torn fpstate during enable.
- **Confidential FPU (SEV-ES/TDX) read-protected** — defense against host reading encrypted state.
- **Per-vCPU last_cpu tracking** — defense against stale per-CPU FPU assumption on cross-CPU migrate.
- **MPX features disabled by default** (deprecated) — defense against ISA-erratum.
- **XSAVES vs XSAVE selection** — defense against incorrect compaction format on confidential FPU.
- **Per-XSAVE component dependency check** — defense against AVX-without-SSE state-corruption.
- **Live-migrate XSAVE2 ioctl includes all features** — defense against post-migrate state truncation.
- **Per-CPU host FPU restored before kernel-fpu-begin** — defense against host kernel-FPU-task corruption.

