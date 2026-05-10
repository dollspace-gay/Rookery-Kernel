# Tier-3: arch/x86/kvm/vmx/nested.c — Nested-VMX (L1 KVM-on-KVM running L2; vmcs12 emulation + VMCS-shadowing accel)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-vmx.md
upstream-paths:
  - arch/x86/kvm/vmx/nested.c
  - arch/x86/kvm/vmx/nested.h
  - arch/x86/kvm/vmx/vmcs12.c
  - arch/x86/kvm/vmx/vmcs12.h
  - arch/x86/kvm/vmx/vmcs_shadow_fields.h
-->

## Summary

Nested-VMX lets L1 (a guest of L0=KVM) itself run KVM and host its own L2 guests. Per L1-VMRESUME, L0 KVM intercepts and: copies L1's vmcs12 (per-Intel-spec layout) to L0's actual vmcs02 (the real CPU VMCS), runs L2 via VMRESUME-on-vmcs02, and copies vmexit-state back to L1's vmcs12 on L2-vmexit. VMCS-shadowing is an Intel-CPU feature that lets L1 directly read/write a subset of vmcs12 fields without VM-exit (KVM marks fields shadowed via vmcs_shadow_fields.h). Massively complex (~7456 lines) — biggest VMX-specific file after vmx.c itself. Critical for: KVM-on-KVM dev/test stacks, cloud-nested-virt, anti-cheat detection.

This Tier-3 covers `nested.c` (~7456 lines) + `nested.h` + `vmcs12.c` + `vmcs12.h` + `vmcs_shadow_fields.h`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vmcs12` | per-L1 VMCS-12 ABI struct | `kernel::kvm::x86::vmx::nested::Vmcs12` |
| `struct nested_vmx` | per-L1-vCPU nested state | `VmxNested` |
| `struct vmcs_host_state` (extended for nested) | per-L1 host-state buffer | shared |
| `nested_vmx_init_vcpu(vcpu)` | per-L1-vCPU nested init | `VmxNested::init_vcpu` |
| `nested_vmx_free_vcpu(vcpu)` | per-L1-vCPU teardown | `VmxNested::free_vcpu` |
| `nested_vmx_run(vcpu, launch)` | VMRESUME / VMLAUNCH from L1 | `VmxNested::run` |
| `nested_vmx_vmexit(vcpu, exit_reason, exit_intr_info, exit_qualification)` | L2-vmexit handling | `VmxNested::vmexit` |
| `prepare_vmcs02(vcpu, vmcs12)` | copy vmcs12 → vmcs02 pre-launch | `VmxNested::prepare_vmcs02` |
| `prepare_vmcs12(vcpu, vmcs12, exit_reason, ...)` | copy vmcs02 → vmcs12 post-vmexit | `VmxNested::prepare_vmcs12` |
| `nested_vmx_check_vmcs12(vcpu)` | per-launch validate vmcs12 fields | `VmxNested::check_vmcs12` |
| `vmx_check_msr_bitmap_addr(...)` | per-MSR-bitmap validation | `VmxNested::check_msr_bitmap_addr` |
| `nested_vmx_handle_eptp_setup(...)` | L2 EPT pointer setup | `VmxNested::handle_eptp_setup` |
| `nested_vmx_msr_*` | per-MSR access (vmcs-fields, ENLIGHTENED_VMCS, etc.) | `VmxNested::msr_*` |
| `vmread_from_vmcs01_or_vmcs12(...)` | per-field VMREAD dispatch | `VmxNested::vmread` |
| `vmwrite_to_vmcs01_or_vmcs12(...)` | per-field VMWRITE dispatch | `VmxNested::vmwrite` |
| `handle_vmread(vcpu)` (vmx.c) | guest VMREAD intercept | `VcpuVmx::handle_vmread` |
| `handle_vmwrite(vcpu)` | guest VMWRITE intercept | `VcpuVmx::handle_vmwrite` |
| `handle_vmlaunch(vcpu)` / `_vmresume(vcpu)` | guest VMLAUNCH/VMRESUME intercept | `VcpuVmx::handle_vmlaunch` / `_vmresume` |
| `handle_vmclear(vcpu)` / `_vmptrld(vcpu)` / `_vmptrst(vcpu)` | per-VMCS pointer ops | `VcpuVmx::handle_vmclear` / `_vmptrld` / `_vmptrst` |
| `nested_vmx_setup_ctls_msrs(...)` | per-VM caps advertised to L1 | `VmxNested::setup_caps` |
| `nested_change_vmcs01_vpid_to_vmcs02_vpid(...)` | per-L2 VPID assignment | `VmxNested::change_vpid` |
| `vmcs_shadow_fields[]` (vmcs_shadow_fields.h) | per-field shadow-eligibility list | `VmcsShadow::FIELDS` |

## Compatibility contract

REQ-1: Per-L1-vCPU `nested_vmx`:
- `vmxon` (bool: L1 has VMXON-ed).
- `current_vmptr` (L1's currently-loaded VMCS pointer; guest-PA).
- `pi_desc` (L2 posted-IRQ descriptor).
- `vmcs12_cache` (per-L1-vCPU latest read of vmcs12 for fast-path).
- `vmcs02` (L0's actual VMCS used to run L2; L1-vCPU dedicated).
- `nested_run_pending` (L2 about to run on next vmenter).
- `enlightened_vmcs` (Hyper-V evmcs accel state).
- `posted_intr_nv` (notification vector).

REQ-2: vmcs12 ABI (1KiB struct):
- Identical layout to `struct vmcs12` in vmcs12.h.
- Used by L1 for VMREAD/VMWRITE; L0 reads on launch + writes on exit.
- Fields cover: control area + read-only data area + guest-state + host-state.

REQ-3: VMXON / VMXOFF (handle_vmxon / _vmxoff):
- L1 executes VMXON instruction → vmexit → KVM emulates.
- Validate L1 CR0/CR4 + arg pointer.
- Allocate vmcs02; init nested.vmxon = true.
- VMXOFF: clear nested.vmxon; clean up.

REQ-4: VMPTRLD (load vmcs12 pointer):
- L1 specifies guest-PA of its vmcs12.
- KVM reads first 32 bits (revision-id); validates matches.
- nested.current_vmptr = guest-PA.

REQ-5: VMCLEAR (clear active vmcs12):
- VMCS12 marked launched=false; subsequent VMRESUME will fail (must VMLAUNCH).

REQ-6: VMLAUNCH / VMRESUME flow:
1. L1 executes VMLAUNCH (or VMRESUME) → L0 vmexit.
2. handle_vmlaunch():
   - nested_vmx_check_vmcs12(vcpu) — per-field validate.
   - prepare_vmcs02(vcpu, vmcs12) — copy fields to vmcs02:
     - control area: from vmcs12 + L0-mandated bits.
     - guest-state: from vmcs12 (L2 state).
     - host-state: from vmcs01 (L1 state for L2-vmexit).
     - EPTP: nested EPTP pointing to L0's nested-page-table mapping L2 GPA → L1 GPA → host PA.
   - VMRESUME on vmcs02 → L2 runs on real CPU.
3. L2 vmexit fires.
4. nested_vmx_vmexit(vcpu, exit_reason, intr_info, qual):
   - prepare_vmcs12(vcpu, vmcs12, exit_reason, ...) — copy vmcs02 → vmcs12.
   - Switch back to vmcs01.
   - Inject vmexit info to L1 via vmcs01 fields.
5. L1 continues; sees its own VMRESUME returned with exit-info populated.

REQ-7: VMREAD / VMWRITE:
- L1 issues VMREAD/VMWRITE on its vmcs12.
- L0 intercepts via vmexit; emulates by reading/writing vmcs12 in guest memory.
- For shadowed fields (per vmcs_shadow_fields.h): L0 enables VMCS-shadowing in vmcs01; L1 directly reads/writes without vmexit.

REQ-8: VMCS-shadowing acceleration:
- VMCS12 fields shadowable per Intel-spec (specific fields like VM_INSTRUCTION_ERROR, EXIT_QUALIFICATION, etc.).
- L0 allocates "shadow VMCS" (separate phys page); points vmcs01.VMREAD-bitmap + VMWRITE-bitmap to mark shadowed-field bits.
- L1 VMREAD/VMWRITE on shadowed-field directly hits CPU-shadow-VMCS without exit.
- L0 syncs shadow ↔ real vmcs12 at vmenter/vmexit boundary.

REQ-9: Nested EPTP:
- L1's vmcs12.EPTP points to L1-managed page-table mapping L2-GPA → L1-GPA.
- L0 wraps with its own page-walk to map L1-GPA → host-PA.
- Result: L2-GPA → L1-GPA → host-PA via nested-page-walk.
- KVM TDP-MMU handles the L0 layer; vmcs02.EPTP points to KVM's combined mapping.

REQ-10: Per-L2 VPID:
- L0 assigns unique VPID for L2; vmcs02.VPID populated.
- INVVPID dispatches per-L2-VPID for selective TLB-flush.

REQ-11: Posted-IRQ for L2:
- Per-L1-vCPU posted-IRQ descriptor for L2-vCPU.
- L0 redirects host-IPI to L2 via PI-desc.

REQ-12: Hyper-V Enlightened VMCS (evmcs):
- Hyper-V-on-KVM L1 uses evmcs (Hyper-V-defined VMCS-shadowing alternative).
- KVM emulates evmcs accesses; shadow optimizes.

REQ-13: Per-VM nested-VMX caps (`nested_vmx_setup_ctls_msrs`):
- VMX MSRs (MSR_IA32_VMX_*) advertised to L1 reflect L0's actual + KVM's nested-cap policy.
- L1 reads CAPS to know what VMX features it can use.

REQ-14: Live migration of nested guest:
- Pre-migrate: KVM_SET_NESTED_STATE / _GET_NESTED_STATE for full nested state.
- Per-L2 state preserved across migrate.

## Acceptance Criteria

- [ ] AC-1: Boot L1 with `-cpu kvm64,vmx=on`: L1 sees VMX in CPUID.
- [ ] AC-2: L1 runs KVM module: kvm_intel loads inside L1.
- [ ] AC-3: L1 boots L2 guest (KVM-on-KVM): L2 boots Linux distro to login prompt.
- [ ] AC-4: VMCS-shadowing test: VMREAD-stress in L1; vmexit count reduced ~95% vs no-shadowing.
- [ ] AC-5: Multi-L2: L1 hosts 4 L2 guests concurrently; all run.
- [ ] AC-6: KVM_GET_NESTED_STATE / _SET_NESTED_STATE round-trip preserves state.
- [ ] AC-7: Spec-violation defense: malformed vmcs12 (invalid CR0 PG=1 PE=0) at VMLAUNCH causes VMfailValid to L1.
- [ ] AC-8: Hyper-V evmcs: KVM running on Hyper-V using evmcs path; reduced exit count.
- [ ] AC-9: kvm-unit-tests `nested_vmx` test passes.

## Architecture

`VmxNested` per-L1-vCPU:

```
struct VmxNested {
  vmxon: bool,
  current_vmptr: u64,
  current_vmcs12: KArc<Vmcs12>,                 // L1's vmcs12 (host-mapped)
  vmcs02: KBox<Vmcs>,                            // actual CPU VMCS for L2
  vmcs02_pa: u64,
  shadow_vmcs: Option<KBox<Vmcs>>,               // shadow accel
  shadow_vmcs_pa: u64,
  nested_run_pending: bool,
  preemption_timer_expired: bool,
  vmexit_msr_load: KBox<MsrAutoload>,
  vmexit_msr_store: KBox<MsrAutoload>,
  vmentry_msr_load: KBox<MsrAutoload>,
  pi_desc: KBox<PostedIntrDesc>,
  pi_desc_addr: u64,
  posted_intr_nv: u8,
  enlightened_vmcs: Option<KArc<HvEvmcs>>,
  smm: SmmContext,
  nested_vmx_msrs: NestedVmxMsrs,
}
```

`VmxNested::run(vcpu, launch)` flow:
1. nested_vmx_check_vmcs12(vcpu) → if invalid: emulate VMfailValid; return.
2. prepare_vmcs02(vcpu, &vmcs12):
   - Acquire vmcs12 contents from guest memory.
   - Copy control area: pin-based + cpu-based + secondary + tertiary controls (intersect with L0-allowed).
   - Copy guest-state: CR0/CR3/CR4/EFER/segments/IDTR/GDTR/RIP/RSP/RFLAGS/etc.
   - Set host-state in vmcs02 to L1's current state (so L2-vmexit returns to L1).
   - EPTP: nested-EPTP via TDP MMU.
3. vmptrld(vmcs02_pa).
4. vmresume / vmlaunch → L2 runs on CPU.
5. L2 vmexit → control returns.
6. nested_vmx_vmexit dispatched.

`VmxNested::vmexit(vcpu, exit_reason, intr_info, qual)`:
1. prepare_vmcs12(vcpu, vmcs12, exit_reason, ...):
   - Read vmcs02 read-only data area; populate vmcs12.exit_reason + .exit_qualification + .vmexit_intr_info etc.
   - Read vmcs02 guest-state; populate vmcs12.guest-state (L2 state at vmexit).
2. vmptrld(vmcs01_pa) — switch back to L1's VMCS.
3. Re-inject as L1-vmexit via vmcs01.exit_reason field.
4. Continue running L1.

`VmxNested::prepare_vmcs02(vcpu, vmcs12)`:
1. Per-control-field: vmcs02.field = vmcs12.field & L0-supported-mask + L0-mandatory-bits.
2. Per-guest-state-field: vmcs02.field = vmcs12.field.
3. Per-host-state-field: vmcs02.field = vmcs01-current-host-state (L1's).
4. EPTP: compute nested-EPTP.
5. MSR auto-load/store lists: per-vmcs12 array; validate per-MSR.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vmcs12_field_validation` | INVARIANT | per-launch validate vmcs12 fields against Intel-spec; defense against L1-injected invalid state crashing CPU. |
| `vmcs02_isolation` | INVARIANT | per-L1-vCPU vmcs02 distinct from vmcs01; defense against cross-VMCS state pollution. |
| `eptp_chain_bounded` | INVARIANT | nested EPTP walk bounded by 4 levels; defense against unbounded recursion. |
| `vpid_unique_per_l2` | INVARIANT | per-L2 VPID distinct per-L1-vCPU; defense against TLB-aliasing across L2s. |
| `posted_intr_desc_aligned` | INVARIANT | per-L2 PI-desc 64-byte aligned. |

### Layer 2: TLA+

`virt/kvm/nested_vmx_run.tla`:
- Per-L1-vCPU state ∈ {L1Running, EnteringL2, L2Running, ExitingL2, L1Resumed}.
- Properties:
  - `safety_no_l2_without_vmptrld` — L2Running implies vmcs02 vmptrld'ed.
  - `safety_vmexit_reflects_to_l1` — ExitingL2 transitions to L1Resumed with vmcs12 populated.
  - `liveness_l2_eventually_exits` — L2 eventually vmexits.

`virt/kvm/vmcs_shadow_field.tla`:
- Per-field state ∈ {ShadowedRO, ShadowedRW, NotShadowed}.
- Properties:
  - `safety_shadow_consistency` — shadowed field values match vmcs12 at vmenter.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VmxNested::run` post: vmcs02 launched; nested_run_pending == false; L2 executing | `VmxNested::run` |
| `VmxNested::vmexit` post: vmcs12 populated with exit-state; vmcs01 active again | `VmxNested::vmexit` |
| `VmxNested::prepare_vmcs02` post: vmcs02 controls intersect L0-allowed + L1-requested | `VmxNested::prepare_vmcs02` |
| `VmxNested::check_vmcs12` post: returns OK iff all fields valid per spec | `VmxNested::check_vmcs12` |
| Per-L2 EPTP validates pointing to valid memslot range | `VmxNested::handle_eptp_setup` |

### Layer 4: Verus/Creusot functional

`L1 VMRESUME → L2 execution → L2 vmexit → L1 sees vmcs12 populated with exit-info` round-trip equivalence: per-L2-vmexit the L1's vmcs12 contents match what L2 hardware actually saved (modulo emulation differences in unsupported fields).

## Hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

nested-VMX-specific reinforcement:

- **Per-VMRESUME vmcs12 validation** — defense against L1-injected invalid state crashing CPU on real-VMRESUME.
- **L0 intersect-with-supported for control bits** — defense against L1 enabling features L0 cannot guarantee.
- **Nested EPTP walk-depth bounded** — defense against unbounded TDP walk on pathological L1-supplied EPTP.
- **Per-L2 VPID distinct** — defense against TLB-leak across L2 instances.
- **Shadow-VMCS field-bitmap restrictive** — defense against shadowed-field set including write-sensitive fields.
- **Per-L1-vCPU vmcs02 distinct** — defense against vmcs02 reuse across L1 vCPUs causing state corruption.
- **Per-L2 EPT root invalidate on L2-shutdown** — defense against TDP-stale-mapping for next L2 instance.
- **Per-vmcs12 host-state from vmcs01** — defense against L1 setting vmcs12.host-state to malicious values escape on L2-vmexit.
- **MSR-load list bounds checked** — defense against L1 specifying long list causing per-vmexit slowdown.
- **VMfailValid vs VMfailInvalid** per-spec correctness — defense against L1 mis-interpreting our error code.
- **Nested KVM_GET_NESTED_STATE serialization complete** — defense against migration losing L2 state.
- **Per-cycle vmcs02 VMCLEAR before reuse** — defense against per-CPU VMCS-cache stale referencing freed memory.
- **Posted-IRQ NV (notification vector)** validated — defense against L1 specifying reserved vector causing CPU exception.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Nested-SVM (covered in `virt/kvm/x86-nested-svm.md` future Tier-3)
- TDX (covered in `virt/kvm/x86-tdx.md` future Tier-3)
- VMX core (covered in `x86-vmx.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- Hyper-V evmcs full details (covered in `x86-hyperv.md` Tier-3)
- L3+ deep nesting (out-of-spec; not supported)
- Implementation code
