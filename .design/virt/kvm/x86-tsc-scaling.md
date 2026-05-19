# Tier-3: arch/x86/kvm/x86.c (TSC scaling subset) — KVM TSC frequency scaling + offset for live migration

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/x86.c (TSC-scaling at ~2533, ~2659, ~2669, ~2730, ~2790, ~2834, ~2901)
  - arch/x86/kvm/vmx/vmx.c (TSC-multiplier MSR write)
  - arch/x86/kvm/svm/svm.c (TSC-ratio MSR write)
  - arch/x86/include/asm/msr-index.h (MSR_AMD64_TSC_RATIO 0xC0000104; MSR_IA32_TSC_ADJUST 0x3B)
-->

## Summary

Per-vCPU TSC scaling (Intel VT-x SDM-Vol3 §25.5.1 / AMD APM-Vol2 §15.30) virtualizes guest-RDTSC to scale + offset host-TSC: `guest_tsc = (host_tsc × multiplier) >> N + offset`. Critical for: live-migrating a VM between hosts with different TSC frequencies — guest sees consistent virtual_tsc_khz across migrate. KVM tracks per-vCPU `l1_tsc_scaling_ratio` (multiplier) + `l1_tsc_offset` (offset). Per-nested L2 tracks `tsc_scaling_ratio` = L1_ratio × L2_ratio (composed). Per-WRMSR(IA32_TSC) recomputes offset to make guest-RDTSC return target-TSC.

This Tier-3 covers TSC-scaling subset of `x86.c` (~150 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `vcpu->arch.virtual_tsc_khz` | per-vCPU virtual TSC freq | `Vcpu::arch::virtual_tsc_khz` |
| `vcpu->arch.l1_tsc_scaling_ratio` | per-vCPU L1 multiplier | `Vcpu::arch::l1_tsc_scaling_ratio` |
| `vcpu->arch.tsc_scaling_ratio` | per-vCPU effective multiplier (L1 or L1×L2) | `Vcpu::arch::tsc_scaling_ratio` |
| `vcpu->arch.l1_tsc_offset` | per-vCPU L1 TSC offset | `Vcpu::arch::l1_tsc_offset` |
| `kvm_caps.default_tsc_scaling_ratio` | per-VM-default = 1.0 << frac_bits | `KvmCaps::default_tsc_scaling_ratio` |
| `kvm_caps.tsc_scaling_ratio_frac_bits` | VMX:48 / SVM:32 | `KvmCaps::tsc_scaling_ratio_frac_bits` |
| `kvm_caps.max_tsc_scaling_ratio` | VMX:full-u64 / SVM:max | `KvmCaps::max_tsc_scaling_ratio` |
| `kvm_scale_tsc(tsc, ratio)` | per-vCPU scaling fn | `Vcpu::scale_tsc` |
| `kvm_compute_l1_tsc_offset(vcpu, target)` | per-vCPU recompute offset | `Vcpu::compute_l1_tsc_offset` |
| `kvm_calc_nested_tsc_multiplier(l1_m, l2_m)` | compose L1×L2 | `Vcpu::calc_nested_tsc_multiplier` |
| `kvm_calc_nested_tsc_offset(l1_o, l2_o, l2_m)` | compose offsets | `Vcpu::calc_nested_tsc_offset` |
| `kvm_set_tsc_khz(vcpu, user_tsc_khz)` | userspace sets virtual freq | `Vcpu::set_tsc_khz` |
| `vcpu->arch.last_tsc_khz` | per-VM-last virtual freq | `KvmArch::last_tsc_khz` |
| `MSR_IA32_TSC_ADJUST` (0x3B) | per-vCPU TSC adjust MSR | UAPI |
| `MSR_AMD64_TSC_RATIO` (0xC0000104) | AMD TSC scaling MSR | UAPI |

## Compatibility contract

REQ-1: Per-vCPU TSC scale fn:
- `scale_tsc(tsc, ratio) = (tsc × ratio) >> tsc_scaling_ratio_frac_bits`.
- Per-VMX: frac_bits = 48; ratio = guest_tsc_khz × (1 << 48) / host_tsc_khz.
- Per-SVM: frac_bits = 32; ratio = guest_tsc_khz × (1 << 32) / host_tsc_khz.

REQ-2: Per-vCPU effective TSC formula:
- `vcpu_observed_tsc = scale_tsc(host_tsc, ratio) + offset`.
- Where ratio = vcpu.tsc_scaling_ratio (effective: L1 or L1×L2).

REQ-3: Per-vCPU init reset:
- vcpu.l1_tsc_scaling_ratio = kvm_caps.default_tsc_scaling_ratio (1.0).
- vcpu.tsc_scaling_ratio = same.
- vcpu.virtual_tsc_khz = host_tsc_khz.
- vcpu.l1_tsc_offset = 0.

REQ-4: KVM_SET_TSC_KHZ ioctl path:
- userspace sets vcpu.virtual_tsc_khz = user_tsc_khz.
- If host_tsc_khz == user_tsc_khz: ratio = default; no scaling.
- Else: ratio = mul_u64_u32_div(1 << frac_bits, user_tsc_khz, host_tsc_khz).
- Validate: ratio ≠ 0; ratio < kvm_caps.max_tsc_scaling_ratio.
- vcpu.l1_tsc_scaling_ratio = ratio.

REQ-5: WRMSR(IA32_TSC, target):
- Per-vCPU recompute l1_tsc_offset = target - scale_tsc(rdtsc(), l1_ratio).
- Per-VM common-offset detection: if all vCPUs at same khz + last write within tolerance: share offset.
- Per-write logged in kvm.arch.last_tsc_write / last_tsc_khz / last_tsc_nsec.

REQ-6: Nested L2 composition:
- L1 advertises L2 with l2_multiplier (per-VMX vmcs.tsc_multiplier; per-SVM vmcb.tsc_ratio).
- vcpu.tsc_scaling_ratio = mul_u64_u64_shr(l1_multiplier, l2_multiplier, frac_bits).
- vcpu.tsc_offset_when_l2_active = l1_offset + scale(l2_offset, l1_ratio).

REQ-7: Per-vCPU MSR_IA32_TSC_ADJUST:
- Per-vCPU u64 cumulative adjustment.
- Per-WRMSR(IA32_TSC, T) updates IA32_TSC_ADJUST by (T - current_tsc).
- Per-RDMSR(IA32_TSC_ADJUST) returns cumulative.

REQ-8: Per-vCPU MSR_AMD64_TSC_RATIO (AMD only):
- Per-vCPU read returns guest-set ratio (default = 1.0 << 32).
- Per-vCPU WRMSR validates ratio < max; updates vmcb.tsc_ratio.

REQ-9: Per-VMX vmcs.tsc-multiplier control:
- vmcs.secondary-controls bit USE_TSC_SCALING = 1.
- vmcs.tsc-multiplier (64 bits) loaded at vmenter.

REQ-10: Per-SVM vmcb.tsc-ratio:
- per-vCPU vmcb.control.tsc_ratio (64 bits).

REQ-11: Live-migration:
- Source: KVM_GET_TSC_KHZ + KVM_GET_MSRS(IA32_TSC, IA32_TSC_ADJUST).
- Destination: KVM_SET_TSC_KHZ + KVM_SET_MSRS(IA32_TSC, IA32_TSC_ADJUST).
- l1_tsc_scaling_ratio recomputed from virtual_tsc_khz / host_tsc_khz.

REQ-12: Per-VM TSC sync:
- vcpu.last_tsc_nsec / last_tsc_write tracked.
- If within KVM_TSC_MAX_TOLERANCE_NSEC (1ms): re-use offset (vCPUs synced).

REQ-13: Per-VM cap KVM_CAP_TSC_CONTROL = 1:
- Userspace can set per-vCPU virtual_tsc_khz independently.

## Acceptance Criteria

- [ ] AC-1: Boot guest on host @ 3.0GHz: guest /proc/cpuinfo shows ~3.0GHz (no scaling).
- [ ] AC-2: KVM_SET_TSC_KHZ(2_000_000): vcpu.l1_tsc_scaling_ratio computed; guest RDTSC scales to 2GHz.
- [ ] AC-3: scale_tsc(host_tsc, default_ratio) == host_tsc.
- [ ] AC-4: WRMSR(IA32_TSC, 0): subsequent RDTSC returns small (offset = -scale_tsc(rdtsc()).
- [ ] AC-5: Per-VM 4 vCPUs same TSC freq: shared l1_tsc_offset.
- [ ] AC-6: Live migration src 3GHz → dest 2GHz: guest sees 3GHz (scaled).
- [ ] AC-7: Nested L1 KVM with L2: vcpu.tsc_scaling_ratio = L1_ratio × L2_ratio.
- [ ] AC-8: WRMSR(IA32_TSC_ADJUST, A) + RDMSR returns A.
- [ ] AC-9: Per-VMX KVM cap KVM_CAP_TSC_CONTROL: returns 1.
- [ ] AC-10: kvm-unit-tests `tsc_scaling` test passes.

## Architecture

Per-vCPU TSC state:

```
struct VcpuArch {
  ...
  virtual_tsc_khz: u32,
  l1_tsc_scaling_ratio: u64,
  tsc_scaling_ratio: u64,                        // l1 or l1×l2
  l1_tsc_offset: i64,
  tsc_offset: i64,                               // l1 or composed
  ia32_tsc_adjust: i64,
}
```

Per-VM TSC tracking:

```
struct KvmArch {
  ...
  last_tsc_khz: u32,
  last_tsc_nsec: u64,
  last_tsc_write: u64,
  cur_tsc_nsec: u64,
  cur_tsc_write: u64,
  cur_tsc_offset: i64,
  cur_tsc_generation: u64,
  nr_vcpus_matched_tsc: AtomicU32,
}
```

Per-KVM caps:

```
struct KvmCaps {
  default_tsc_scaling_ratio: u64,                // VMX: 1<<48; SVM: 1<<32
  tsc_scaling_ratio_frac_bits: u8,               // VMX:48; SVM:32
  max_tsc_scaling_ratio: u64,                    // per-vendor max
  has_tsc_control: bool,
}
```

`Vcpu::scale_tsc(tsc, ratio) -> u64`:
1. If ratio == default_tsc_scaling_ratio: return tsc.
2. Return mul_u64_u64_shr(tsc, ratio, frac_bits).

`Vcpu::compute_l1_tsc_offset(vcpu, target_tsc) -> i64`:
1. tsc_scaled = vcpu.scale_tsc(host_rdtsc(), vcpu.l1_tsc_scaling_ratio).
2. offset = target_tsc - tsc_scaled.
3. Return offset (i64).

`Vcpu::set_tsc_khz(vcpu, user_tsc_khz) -> Result<()>`:
1. vcpu.virtual_tsc_khz = user_tsc_khz.
2. If user_tsc_khz == host_tsc_khz: ratio = default_tsc_scaling_ratio.
3. Else: ratio = mul_u64_u32_div(1u64 << frac_bits, user_tsc_khz, host_tsc_khz).
4. If ratio == 0 or ratio >= max_tsc_scaling_ratio: ratio = default; return Err.
5. vcpu.l1_tsc_scaling_ratio = ratio.
6. vcpu.tsc_scaling_ratio = ratio (until L2 active).
7. vmcs.tsc_multiplier = ratio (vmx); vmcb.tsc_ratio = ratio (svm).

`Vcpu::calc_nested_tsc_multiplier(l1_m, l2_m) -> u64`:
1. If l2_m == default: return l1_m.
2. Return mul_u64_u64_shr(l1_m, l2_m, frac_bits).

`Vcpu::calc_nested_tsc_offset(l1_offset, l2_offset, l2_multiplier) -> i64`:
1. If l2_multiplier == default: return l1_offset + l2_offset.
2. Return l1_offset + scale_tsc(l2_offset, l2_multiplier).

`Vcpu::write_tsc(vcpu, target_tsc)`:
1. offset = vcpu.compute_l1_tsc_offset(target_tsc).
2. vcpu.l1_tsc_offset = offset.
3. vmcs.tsc_offset = offset (or composed for L2).
4. Per-VM: detect synced-vCPUs:
   - If kvm.last_tsc_khz == vcpu.virtual_tsc_khz ∧ |kvm.last_tsc_write - target| < tolerance:
     - kvm.cur_tsc_offset = offset; kvm.nr_vcpus_matched_tsc++.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `scale_tsc_default_passes_through` | INVARIANT | scale_tsc(t, default) == t. |
| `ratio_in_bounds_when_set` | INVARIANT | post-set_tsc_khz: ratio ∈ (0, max_tsc_scaling_ratio). |
| `nested_multiplier_default_idempotent` | INVARIANT | calc_nested(l1, default) == l1. |
| `compute_offset_yields_target` | INVARIANT | post-compute_l1_tsc_offset: scale(host_tsc, ratio) + offset == target. |
| `tsc_scaling_ratio_eq_l1_when_no_l2` | INVARIANT | !l2_active ⟹ vcpu.tsc_scaling_ratio == l1_tsc_scaling_ratio. |

### Layer 2: TLA+

`virt/kvm/tsc_scaling.tla`:
- Per-vCPU virtual_tsc_khz + l1_ratio + l1_offset state.
- Per-write target_tsc → recompute offset.
- Properties:
  - `safety_round_trip_tsc` — per-WRMSR(TSC, T) → next-RDTSC(TSC) ≥ T (monotonic).
  - `safety_scaling_consistent_across_migration` — pre-migrate guest_tsc == post-migrate guest_tsc (within tolerance).
  - `liveness_synced_vcpus_share_offset` — per-VM all vCPUs same khz+target ⟹ same offset.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::set_tsc_khz` post: l1_ratio computed correctly; vmcs/vmcb updated | `Vcpu::set_tsc_khz` |
| `Vcpu::compute_l1_tsc_offset` post: target_tsc reachable | `Vcpu::compute_l1_tsc_offset` |
| `Vcpu::calc_nested_tsc_multiplier` post: composes correctly | `Vcpu::calc_nested_tsc_multiplier` |
| `Vcpu::write_tsc` post: vcpu.l1_tsc_offset adjusted; per-VM tracking updated | `Vcpu::write_tsc` |

### Layer 4: Verus/Creusot functional

`Per-WRMSR(IA32_TSC, target) → vcpu sees ≈ target on RDTSC; per-virtual_tsc_khz changes propagate to guest RDTSC rate` semantic equivalence: per-vCPU TSC scaling matches Intel SDM §25.5.1 + AMD APM §15.30.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

TSC-scaling-specific reinforcement:

- **Per-vCPU ratio bounded** — defense against ratio=0 (divide-by-zero in HW) or ratio > max (HW undefined).
- **Per-frac-bits per-vendor** — defense against using SVM frac (32) on VMX (48) causing wrong scale.
- **Per-vCPU virtual_tsc_khz validated** — defense against insane khz.
- **Per-VM synced-vCPU detection** — defense against per-vCPU offset drift.
- **Per-vCPU vmcs.tsc_multiplier loaded** — defense against post-vmenter wrong scale.
- **Per-vCPU vmcb.tsc_ratio loaded** — defense against AMD post-vmrun wrong scale.
- **Per-WRMSR(IA32_TSC) audit** — defense against guest manipulating IA32_TSC_ADJUST drift.
- **Per-live-migration: l1_ratio recomputed** — defense against post-migrate wrong scale.
- **Per-nested L1×L2 multiplication overflow checked** — defense against ratio overflow.
- **Per-vCPU offset i64-typed** — defense against unsigned-wrap on negative offsets.
- **Per-VM cap KVM_CAP_TSC_CONTROL gated by vendor support** — defense against userspace requesting unsupported feature.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — KVM_GET/SET_TSC_KHZ and KVM_SET_TSC_KHZ_PER_VM bounded by `__u32` payload; no user-pointer copy in the scaling computation.
- **PAX_KERNEXEC** — `kvm_compute_tsc_offset_l1`, `kvm_scale_tsc`, vendor `write_tsc_multiplier` resolve through RO `kvm_x86_ops` vtable; module text RX-only.
- **PAX_RANDKSTACK** — WRMSR(TSC) and VMRUN re-entry inherit RANDKSTACK.
- **PAX_REFCOUNT** — per-VM `max_tsc_khz`, per-vCPU `l1_tsc_scaling_ratio` saturating refcount.
- **PAX_MEMORY_SANITIZE** — per-vCPU TSC scaling state (tsc_scaling_ratio, l1_tsc_scaling_ratio, virtual_tsc_mult) zeroed on alloc/free.
- **PAX_UDEREF** — KVM_SET_TSC_KHZ copy_from_user STAC/CLAC bracketed.
- **PAX_RAP / kCFI** — `kvm_x86_ops.write_tsc_multiplier`, `write_tsc_offset` slots RAP-signed.
- **GRKERNSEC_HIDESYM** — per-vCPU scaling ratio kaddr redacted unless CAP_SYSLOG + gr-rbac.
- **GRKERNSEC_DMESG** — TSC unstable / scaling overflow warnings rate-limited and gated by `dmesg_restrict`.
- **KVM ioctl CAP_SYS_ADMIN strict** — KVM_CAP_TSC_CONTROL / KVM_SET_TSC_KHZ gated to CAP_SYS_ADMIN + gr-rbac VM-create role.
- **TSC-scaling bounded** — multiplier capped at hardware `kvm_max_tsc_scaling_ratio`; guest WRMSR cannot drive ratio overflow.
- **VMCS/VMCB tsc_multiplier loaded pre-VMRUN** — `vmcs.tsc_multiplier` / `vmcb.tsc_ratio` populated before VMRUN so post-entry guest sees correct scale.
- **L1×L2 nested-scaling overflow checked** — nested combined ratio (l1_tsc_scaling × l2_tsc_scaling) checked for overflow before VMCS12 load.
- **Offset i64-typed** — TSC offset signed 64-bit so negative offsets (live-migration backwards adjustment) do not wrap.
- **KVM_CAP_TSC_CONTROL gated by vendor support** — userspace requesting scaling on a non-supporting CPU rejected at capability query.

Per-doc rationale: TSC scaling lets KVM make a slow host appear as a faster TSC to the guest; the grsec reinforcement caps the multiplier to prevent kernel-side overflow, validates nested L1×L2 combined ratio, and SANITIZEs per-vCPU scaling state across migrations so a destroyed vCPU's scaling ratio cannot bleed into a successor.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- KVM RDTSC handling (covered in `x86-rdtsc.md` Tier-3)
- KVM TSC-deadline timer (covered in `x86-tsc-deadline.md` Tier-3)
- KVM PVCLOCK / KVMCLOCK (covered in `x86-pvclock.md` Tier-3)
- Per-arch host TSC initialization (covered separately)
- Implementation code
