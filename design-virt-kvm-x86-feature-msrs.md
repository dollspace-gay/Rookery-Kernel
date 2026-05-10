---
title: "Tier-3: arch/x86/kvm/x86.c (feature-MSR subset) — KVM feature-control MSR virtualization (MSR_IA32_ARCH_CAPABILITIES + MSR_IA32_FEAT_CTL + MSR_IA32_TSX_CTRL + MSR_IA32_FLUSH_CMD)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

x86 has a set of "feature-control" MSRs that gate CPU-side opt-in to specific extensions: `MSR_IA32_FEAT_CTL` controls VMX-enable + SGX-enable + LMCE-enable lock-bit; `MSR_IA32_ARCH_CAPABILITIES` advertises CPU-side feature/erratum capabilities (RDCL_NO, IBRS_ALL, RSBA, SKIP_L1DFL_VMENTRY, SSB_NO, MDS_NO, IF_PSCHANGE_MC_NO, TSX_CTRL, etc.); `MSR_IA32_TSX_CTRL` controls TSX/HLE-disable per RTM/HLE bits; `MSR_IA32_FLUSH_CMD` per-CPU L1D-cache flush. KVM virtualizes by per-vCPU shadow + per-VM mask intersect host-cap with guest-CPUID. Critical for: Spectre/Meltdown mitigation policy + SGX/VMX capability advertising to guest.

This Tier-3 covers feature-MSR handling in `arch/x86/kvm/x86.c` (~200-300 lines).

### Acceptance Criteria

- [ ] AC-1: Boot Linux guest: cat /proc/cpuinfo | grep -E "rdcl_no|ibrs|ssbd": flags reflect host arch_caps.
- [ ] AC-2: WRMSR(MSR_IA32_FEAT_CTL, VMX|LOCK): guest VMX enabled (with VMX-cap CPUID).
- [ ] AC-3: TSX disable: WRMSR(MSR_IA32_TSX_CTRL, RTM_DISABLE); guest XBEGIN traps as #UD.
- [ ] AC-4: L1D flush: WRMSR(MSR_IA32_FLUSH_CMD, 1); per-CPU L1D flushed.
- [ ] AC-5: IBRS pass-through: WRMSR(MSR_IA32_SPEC_CTRL, IBRS); subsequent reads return IBRS.
- [ ] AC-6: IBPB write: WRMSR(MSR_IA32_PRED_CMD, IBPB); branch-predictor flushed.
- [ ] AC-7: Live migration: per-vCPU feature MSRs preserved.
- [ ] AC-8: Per-VM mask: userspace masks RDCL_NO; guest sees vulnerability flag set.
- [ ] AC-9: KVM_X86_GET_MSR_FEATURE_INDEX_LIST: returns feature MSRs.
- [ ] AC-10: kvm-unit-tests `feature_msrs` test passes.

### Architecture

`Vcpu::get_arch_capabilities()`:
1. data := kvm_host.arch_capabilities.
2. Per-CPU additional masking based on CPU-vendor / model.
3. Filter unsupported bits.
4. Return data.

`Vcpu::handle_msr` MSR_IA32_ARCH_CAPABILITIES branch:
- WRMSR (host_initiated only): vcpu.arch.arch_capabilities = data & host_supported_mask.
- WRMSR (guest_initiated): rejected (read-only).
- RDMSR: return vcpu.arch.arch_capabilities.

`Vcpu::handle_msr` MSR_IA32_FEAT_CTL branch:
- WRMSR (guest):
  - If LOCK already set: -EINVAL.
  - Validate flag combinations:
    - VMX_OUTSIDE_SMX requires VMX-cap.
    - SGX_LC_ENABLE requires SGX-cap.
  - vcpu.arch.feat_ctl = data.
- RDMSR: return vcpu.arch.feat_ctl.

`Vcpu::handle_msr` MSR_IA32_TSX_CTRL branch:
- WRMSR: vcpu.arch.tsx_ctrl = data.
  - If RTM_DISABLE: subsequent guest TSX (XBEGIN/XEND) raises #UD.
  - If TSX_CPUID_CLEAR: clear TSX bits in CPUID return.
- RDMSR: return vcpu.arch.tsx_ctrl.

`Vcpu::handle_msr` MSR_IA32_FLUSH_CMD branch:
- WRMSR: if data & L1D_FLUSH:
  - kvm_x86_call(flush_l1d)(vcpu) — per-CPU L1D flush.
- RDMSR: -EINVAL (write-only).

`Vcpu::handle_msr` MSR_IA32_SPEC_CTRL branch:
- WRMSR: vcpu.arch.spec_ctrl = data.
  - msr_autoload set MSR_IA32_SPEC_CTRL → guest's value at vmenter.
- RDMSR: return vcpu.arch.spec_ctrl.

`Vcpu::handle_msr` MSR_IA32_PRED_CMD branch:
- WRMSR: if data & IBPB:
  - kvm_x86_call(handle_ibpb)(vcpu).
- RDMSR: -EINVAL.

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- KVM PMU / PEBS (covered in `x86-pmu.md` / `x86-vmx-pebs.md` Tier-3s)
- KVM_X86_SET_MSR_FILTER (covered in `x86-msr-filter.md` Tier-3)
- VMX core (covered in `x86-vmx.md` Tier-3)
- Spectre/Meltdown mitigations userspace policy (covered in `arch/x86/cpu/bugs.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_get_arch_capabilities()` | per-host arch_caps query | `Vcpu::get_arch_capabilities` |
| `kvm_host.arch_capabilities` | per-host snapshot | `KvmHost::arch_capabilities` |
| `MSR_IA32_ARCH_CAPABILITIES` (0x10A) | guest-visible arch-caps | UAPI |
| `MSR_IA32_FEAT_CTL` (0x3A) | feature-control MSR | UAPI |
| `MSR_IA32_TSX_CTRL` (0x122) | TSX-control | UAPI |
| `MSR_IA32_FLUSH_CMD` (0x10B) | L1D flush command | UAPI |
| `MSR_IA32_DEBUGCTLMSR` (0x1D9) | debug-ctrl with TR + LBR + BTF bits | UAPI |
| `MSR_IA32_PRED_CMD` (0x49) | branch-predictor command (IBPB) | UAPI |
| `MSR_IA32_SPEC_CTRL` (0x48) | speculation-control (IBRS/STIBP/SSBD) | UAPI |
| `MSR_IA32_OVERCLOCKING_STATUS` | per-vCPU OC status | UAPI |
| `MSR_IA32_PERF_CAPABILITIES` (0x345) | perfmon capabilities | UAPI |
| `MSR_AMD64_VIRT_SPEC_CTRL` | AMD virt-spec-ctrl | UAPI |
| `kvm_msr_ignored_check(...)` | per-MSR ignored-msr policy | `Vcpu::msr_ignored_check` |
| `vmx_set_intercept_for_msr(...)` arch-cap branch | per-MSR pass-through | covered in `x86-msr.md` |

### compatibility contract

REQ-1: MSR_IA32_ARCH_CAPABILITIES bits:
- Bit[0] RDCL_NO (Rogue-Data-Cache-Load not vulnerable).
- Bit[1] IBRS_ALL (IBRS effective in user-mode).
- Bit[2] RSBA (RSB Alternate behavior).
- Bit[3] SKIP_L1DFL_VMENTRY (skip L1D-flush at VMENTER).
- Bit[4] SSB_NO (SSB not vulnerable).
- Bit[5] MDS_NO.
- Bit[6] IF_PSCHANGE_MC_NO.
- Bit[7] TSX_CTRL.
- Bit[8] TAA_NO (TSX-Async-Abort not vulnerable).
- Bit[10] MISC_PACKAGE_CTRLS.
- Bit[11] ENERGY_FILTERING_CTL.
- Bit[12] DOITM (Data-Operand-Independent-Timing-Mode).
- Bit[13] SBDR_SSDP_NO.
- Bit[14] FBSDP_NO.
- Bit[15] PSDP_NO.
- Bit[16] FB_CLEAR_DEFAULT.
- Bits[17..23] additional (RFDS_NO, BHI_NO, PBRSB_NO, etc.).

REQ-2: Per-host kvm_host.arch_capabilities:
- Read from MSR at module init.
- Used to compute per-vCPU virtualized arch-caps.

REQ-3: Per-vCPU MSR_IA32_ARCH_CAPABILITIES:
- vcpu.arch.arch_capabilities = host_arch_caps & guest_supported_mask.
- Per-VM masking lets userspace expose subset (e.g., for security model).

REQ-4: MSR_IA32_FEAT_CTL bits:
- Bit[0] LOCK (configuration locked once set).
- Bit[1] VMX_INSIDE_SMX (VMX in SMX root mode).
- Bit[2] VMX_OUTSIDE_SMX (VMX in non-SMX root mode).
- Bit[15..18] SENTER_GLOBAL_ENABLE.
- Bit[18] SGX_LC_ENABLE (SGX Launch-Control).
- Bit[20] LMCE_ON (Local MCE enable).

REQ-5: MSR_IA32_TSX_CTRL bits:
- Bit[0] RTM_DISABLE.
- Bit[1] TSX_CPUID_CLEAR.

REQ-6: MSR_IA32_FLUSH_CMD bits:
- Bit[0] L1D_FLUSH (write-1-to-flush L1D).

REQ-7: Per-vCPU MSR_IA32_SPEC_CTRL bits:
- Bit[0] IBRS (Indirect-Branch Restricted Speculation).
- Bit[1] STIBP (Single-Thread Indirect Branch Predictors).
- Bit[2] SSBD (Speculative-Store-Bypass-Disable).

REQ-8: Per-vCPU MSR_IA32_PRED_CMD:
- Bit[0] IBPB (Indirect-Branch-Predictor-Barrier).
- Write-only; clears branch-predictor cache.

REQ-9: Per-vCPU MSR_IA32_DEBUGCTLMSR:
- Bit[0] LBR (Last-Branch-Record enable).
- Bit[1] BTF (Branch-Trap Flag).
- Bits[6..9] FREEZE_*.
- Bit[14] FREEZE_WHILE_SMM.

REQ-10: Per-vCPU MSR_IA32_PERF_CAPABILITIES:
- Bits[0..7] LBR_FMT (LBR format version).
- Bit[12] FW_WRITES.
- Bit[15] PEBS_BASELINE.
- Bits[16..17] PEBS_FMT.
- Bit[20] PERF_METRICS_AVAILABLE.

REQ-11: Per-VM MSR-feature mask:
- userspace KVM_X86_SET_MSR_FILTER + per-VM CPUID masking control which feature MSRs guest sees.

REQ-12: Per-vCPU SPEC_CTRL pass-through:
- If host CPU has SPEC_CTRL: KVM allows direct guest access (pass-through bitmap).
- Per-vmenter/vmexit: msr_autoload swap host vs guest IBRS.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `arch_caps_mask_subset` | INVARIANT | per-vCPU arch_capabilities ⊆ host arch_capabilities. |
| `feat_ctl_lock_immutable` | INVARIANT | per-vCPU feat_ctl LOCK bit one-way (clear → set; never cleared). |
| `tsx_ctrl_validates_RTM_TSX_CPUID` | INVARIANT | per-vCPU TSX_CTRL bits valid. |
| `spec_ctrl_subset_supported` | INVARIANT | per-vCPU SPEC_CTRL bits ⊆ host-supported (IBRS/STIBP/SSBD). |
| `flush_cmd_write_only` | INVARIANT | per-vCPU FLUSH_CMD WRMSR-only; RDMSR returns -EINVAL. |

### Layer 2: TLA+

`virt/kvm/feature_msr_lifecycle.tla`:
- Per-vCPU per-MSR state: { value, writable, lock_state }.
- Properties:
  - `safety_lock_bit_irrevocable` — once LOCK set, no transition back.
  - `safety_arch_caps_intersect_host` — per-vCPU arch_caps ⊆ host.
  - `liveness_msr_writes_eventually_visible` — guest WRMSR observable via subsequent RDMSR.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::get_arch_capabilities` post: returned bits ⊆ host arch_capabilities | `Vcpu::get_arch_capabilities` |
| MSR_IA32_FEAT_CTL WRMSR post: LOCK bit honored; flag combinations validated | `Vcpu::handle_msr` |
| MSR_IA32_FLUSH_CMD WRMSR post: per-CPU L1D flush triggered iff data&L1D_FLUSH | `Vcpu::handle_msr` |
| MSR_IA32_PRED_CMD WRMSR post: IBPB triggered iff data&IBPB | `Vcpu::handle_msr` |

### Layer 4: Verus/Creusot functional

`Per-vCPU feature MSR access: subsequent RDMSR returns last-written value (modulo intercept policy)` semantic equivalence: per-vCPU per-MSR value tracks WRMSR sequence.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

feature-MSR-specific reinforcement:

- **arch_capabilities masked to host-supported** — defense against guest claiming vulnerable-with-mitigation that host lacks.
- **FEAT_CTL.LOCK irrevocable** — defense against guest unlocking + reconfiguring.
- **FLUSH_CMD per-CPU action validated** — defense against guest abusing flush as DoS.
- **PRED_CMD IBPB rate-limited** — defense against IBPB-flood causing host-CPU performance collapse.
- **SPEC_CTRL pass-through gated** on host SPEC_CTRL feature — defense against guest writing unsupported bits.
- **TSX_CTRL RTM-disable propagates** to guest CPUID — defense against guest using TSX after disable.
- **MSR_IA32_PERF_CAPABILITIES masked** — defense against guest claiming PEBS-fmt host doesn't support.
- **VIRT_SPEC_CTRL (AMD) per-vCPU isolated** — defense against cross-vCPU spec-state leak.
- **Live-migrate per-vCPU feature MSRs validated against destination CPUID** — defense against post-migrate invalid state.
- **FEAT_CTL.SGX_LC_ENABLE gated** — defense against unauthorized SGX-LC config.

