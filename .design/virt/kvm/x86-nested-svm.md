# Tier-3: arch/x86/kvm/svm/nested.c — Nested-SVM (L1 KVM-on-KVM running L2 with VMRUN/VMEXIT VMCB-shadow + nested NPT)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-svm.md
upstream-paths:
  - arch/x86/kvm/svm/nested.c
  - arch/x86/kvm/svm/svm.h (nested fields)
-->

## Summary

Nested-SVM lets L1 (a KVM guest) itself run KVM (or other AMD-SVM hypervisor) and host L2 guests. Per L1-VMRUN of an L2 vmcb, L0 KVM intercepts and: copies L1's vmcb12 (per-AMD-spec layout) into actual vmcb02 used to run L2 on physical CPU, runs L2 via VMRUN-on-vmcb02, and copies L2-vmexit state back to vmcb12 on exit. Simpler than nested-VMX in some ways (no separate VMCS-shadow feature; AMD VMCB has fewer mode-related complexities) but adds nested-NPT: L1's NPT mapping L2-GPA → L1-GPA combined with L0's mapping L1-GPA → host-PA via TDP-MMU. Smaller than nested-VMX (~2083 lines vs 7456).

This Tier-3 covers `arch/x86/kvm/svm/nested.c` (~2083 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vmcb` (re-used) | per-L1 vmcb12 ABI struct | `kernel::kvm::x86::svm::Vmcb` (shared) |
| `struct svm_nested_state` | per-L1-vCPU nested state | `kernel::kvm::x86::svm::nested::SvmNested` |
| `nested_svm_init(vcpu)` | per-L1-vCPU init | `SvmNested::init` |
| `nested_svm_destroy(vcpu)` | teardown | `SvmNested::destroy` |
| `nested_svm_vmrun(vcpu)` | VMRUN from L1 | `SvmNested::vmrun` |
| `nested_svm_vmexit(vcpu)` | L2 exits to L1 | `SvmNested::vmexit` |
| `enter_svm_guest_mode(vcpu, vmcb12_gpa, vmcb12, &arch)` | switch to L2-running | `SvmNested::enter_guest_mode` |
| `nested_svm_load_cr3(vcpu, cr3, nested_npt, ...)` | nested CR3 setup | `SvmNested::load_cr3` |
| `nested_svm_check_intercepts(vcpu, ...)` | per-event intercept check | `SvmNested::check_intercepts` |
| `nested_svm_vmexit_emulated(vcpu)` | L2-vmexit per-event emit to L1 | `SvmNested::vmexit_emulated` |
| `nested_svm_inject_exception(vcpu, vector, error_code)` | exception injection to L2 | `SvmNested::inject_exception` |
| `recalc_intercepts(svm)` | recompute vmcb intercept-bitmap (L0 + L1) | `SvmNested::recalc_intercepts` |
| `nested_svm_get_msrpm_offset(...)` | MSR-protect-bitmap-offset lookup | `SvmNested::msrpm_offset` |
| `nested_svm_check_pmu_msr(...)` | MSR-PMU intercept check | `SvmNested::check_pmu_msr` |
| `svm_check_nested_events(vcpu)` | per-event nested-handling decision | `SvmNested::check_nested_events` |
| `nested_svm_load_state(...)` (KVM_SET_NESTED_STATE) | restore on migrate | `SvmNested::load_state` |
| `nested_svm_save_state(...)` (KVM_GET_NESTED_STATE) | snapshot for migrate | `SvmNested::save_state` |
| `is_guest_mode(vcpu)` | per-vCPU running-L2 query | `Vcpu::is_guest_mode` |

## Compatibility contract

REQ-1: Per-L1-vCPU `svm_nested_state`:
- `vmcb02` (L0's actual VMCB used to run L2; per-L1-vCPU dedicated).
- `vmcb02_pa` (vmcb02 physical addr).
- `hsave` (host-state-save area; saved on VMRUN).
- `vmcb12_gpa` (L1's vmcb12 guest-PA).
- `cur_vmcb12` (cached vmcb12 contents).
- `host_intercept_exceptions` (exceptions L0 must intercept regardless of L1).
- `nested_npt` (nested NPT enabled).
- `intercept_dbreg` (L1's debug-reg intercept bitmap shadow).
- `nested_state_pending` (KVM_REQ_NESTED_STATE).

REQ-2: VMCB12 ABI (1KiB per AMD APM):
- Shared layout with vmcb01 (control area + save area).
- Used by L1 directly (writes to vmcb12 in guest memory); L0 reads on VMRUN, writes on vmexit.
- Unlike vmcs12 (Intel), vmcb12 is normal-page-mapped (no special VMREAD/VMWRITE; just memory ops).

REQ-3: VMRUN intercept (handle_vmrun in svm.c):
1. L1 executes VMRUN with vmcb12_gpa in RAX → vmexit.
2. nested_svm_vmrun(vcpu):
   - Read vmcb12 from guest memory.
   - Validate: vmcb12.intercepts must include VMRUN, etc.
   - Copy vmcb12 → vmcb02 with L0 intercepts merged.
   - hsave = current vmcb01-save-area (so we can restore on L2-vmexit).
   - Set vcpu.guest_mode = true.
   - vmcb_load(vmcb02_pa).
3. Subsequent VMRUN on vmcb02 → L2 runs.

REQ-4: VMEXIT processing:
1. L2 vmexit fires.
2. svm_check_nested_events(vcpu) — decide if L0 handles or routes to L1:
   - If exit_code in L1's vmcb12.intercepts → route to L1.
   - Else → L0 handles (e.g., NPT-fault to TDP-MMU).
3. If route-to-L1: nested_svm_vmexit:
   - Copy vmcb02 read-only data → vmcb12.
   - vmcb_load(vmcb01_pa).
   - Restore vmcb01 from hsave.
   - Inject vmexit-info into L1 via vmcb01.exit_code.

REQ-5: Per-event recalc_intercepts:
- vmcb02.intercepts = vmcb01.intercepts ∪ vmcb12.intercepts.
- Some intercepts mandatory regardless (e.g., #DB intercept for L0 sees all).
- Per-MSR: msrpm_offset(msr) → per-MSR bit in vmcb02.MSRPM = bit in vmcb01.MSRPM | bit in vmcb12.MSRPM.

REQ-6: Nested NPT (Nested Page Table):
- L1 enables NPT in vmcb12.nested_ctl with vmcb12.nested_cr3 → L1-managed NPT mapping L2-GPA → L1-GPA.
- L0 wraps with vmcb02.nested_ctl + vmcb02.nested_cr3 → KVM TDP-MMU mapping L1-GPA → host-PA.
- nested_npt_enabled flag controls whether vmcb02 NPT-walk uses combined or shadow.

REQ-7: Per-L2 ASID:
- L0 assigns unique ASID to L2 (distinct from L1's ASID).
- vmcb02.asid populated per-L2-vCPU.

REQ-8: AVIC (Advanced Virtual Interrupt Controller) with nested:
- L1 may enable AVIC for L2; L0 must coordinate.
- AVIC backing pages tracked nested.

REQ-9: SEV / SEV-ES with nested:
- Nested + SEV requires special handling; out-of-scope at this Tier-3.

REQ-10: Hyper-V on SVM nested:
- Hyper-V running on KVM-AMD; uses Hyper-V-specific paths in svm_onhyperv.c.
- Out-of-scope at Tier-3 detail.

REQ-11: KVM_GET_NESTED_STATE / _SET_NESTED_STATE:
- Per-L1-vCPU snapshot of: nested.vmcb12_gpa, hsave, cur_vmcb12, guest_mode flag.
- Used for live migration of L1-with-running-L2.

REQ-12: Per-VM nested-cap detection:
- AMD CPUID 0x8000000A:EDX nested-virt support gates KVM_CAP_NESTED.
- Without HW support: nested-SVM disabled; L1 sees no SVM CPUID.

## Acceptance Criteria

- [ ] AC-1: Boot L1 with `-cpu kvm64,svm=on`: L1 sees SVM in CPUID.
- [ ] AC-2: L1 runs KVM (kvm-amd module): loads inside L1.
- [ ] AC-3: L1 boots L2: KVM-on-KVM-AMD; L2 boots Linux distro.
- [ ] AC-4: Nested NPT: L2-vCPU page-fault resolved via L1-NPT + L0-NPT chain.
- [ ] AC-5: Multi-L2: L1 hosts 4 L2 guests; all run.
- [ ] AC-6: KVM_GET_NESTED_STATE / _SET_NESTED_STATE round-trip preserves state.
- [ ] AC-7: Per-L2 ASID: distinct ASIDs assigned; INVASID (TLB flush) per-ASID.
- [ ] AC-8: Spec-violation defense: malformed vmcb12 (invalid intercept-set) at VMRUN causes exit-code = INVALID_VMCB.
- [ ] AC-9: kvm-unit-tests `nested_svm` test passes.

## Architecture

`SvmNested` per-L1-vCPU:

```
struct SvmNested {
  vmcb02: KBox<Vmcb>,                          // actual CPU VMCB for L2
  vmcb02_pa: u64,
  hsave: KBox<HostSave>,                       // host-state saved on VMRUN
  vmcb12_gpa: u64,
  cur_vmcb12: KArc<Vmcb>,                      // host-mapped L1's vmcb12
  host_intercept_exceptions: u32,
  nested_npt: bool,
  ctl: SvmNestedCtl,                            // intercept bitmaps
  intercept_cr: u32,
  intercept_dr: u32,
  intercept_exceptions: u32,
  intercept: u64,                               // general intercept
  pause_filter_count: u16,
  pause_filter_thresh: u16,
  iopm_base_pa: u64,
  msrpm_base_pa: u64,
  asid: u32,
}
```

`SvmNested::vmrun(vcpu)` flow:
1. Read vmcb12_gpa from RAX.
2. Map vmcb12 in guest memory; cache contents.
3. Validate vmcb12: intercepts include VMRUN; CR0/CR4 valid; etc.
4. enter_svm_guest_mode(vcpu, vmcb12_gpa, &cur_vmcb12, ...):
   - hsave = vmcb01 save-area.
   - Compute vmcb02 = merge(vmcb01-baseline, vmcb12).
   - vmcb02.intercepts = vmcb01.intercepts ∪ vmcb12.intercepts.
   - vmcb02.save = vmcb12.save (L2 state).
   - vmcb02.nested_ctl = vmcb12.nested_ctl (NPT settings).
   - vmcb02.nested_cr3 = vmcb12.nested_cr3 (or remap via TDP-MMU).
5. vcpu.guest_mode = true.
6. vmcb_load(vmcb02_pa).
7. Subsequent VMRUN on vmcb02 → L2 runs.

`SvmNested::vmexit(vcpu)` flow:
1. Copy vmcb02 read-only fields → vmcb12 (exit_code, exit_info1, exit_info2, exit_int_info).
2. Copy vmcb02 save-area → vmcb12.save (L2 state at vmexit).
3. Restore vmcb01 from hsave.
4. vmcb_load(vmcb01_pa).
5. vcpu.guest_mode = false.
6. Inject vmexit info to L1: vmcb01.exit_code = SVM_EXIT_VMEXIT (re-emit L2-vmexit reason).

`SvmNested::recalc_intercepts(svm)`:
1. vmcb02.intercept_cr = vmcb01.intercept_cr | vmcb12.intercept_cr.
2. vmcb02.intercept_dr = vmcb01.intercept_dr | vmcb12.intercept_dr.
3. vmcb02.intercept_exceptions = vmcb01.intercept_exceptions | vmcb12.intercept_exceptions | mandatory_exceptions.
4. vmcb02.intercept = vmcb01.intercept | vmcb12.intercept.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vmcb12_validation` | INVARIANT | vmcb12 fields validated before VMRUN; defense against L1-injected invalid state. |
| `vmcb02_per_l1_vcpu_unique` | INVARIANT | per-L1-vCPU vmcb02 distinct from vmcb01 and other L1-vCPUs' vmcb02. |
| `nested_npt_walk_bounded` | INVARIANT | nested NPT walk bounded by 4 levels. |
| `asid_unique_per_l2` | INVARIANT | per-L2 ASID distinct from L1 ASID; defense against TLB-aliasing. |
| `hsave_atomic` | INVARIANT | vmcb01 snapshot to hsave atomic; defense against partial-save on irq. |

### Layer 2: TLA+

`virt/kvm/nested_svm_run.tla`:
- Per-L1-vCPU state ∈ {L1Running, EnteringL2, L2Running, ExitingL2, L1Resumed}.
- Properties:
  - `safety_no_l2_without_vmcb02` — L2Running implies vmcb02 vmcb_load'ed.
  - `safety_l2_vmexit_reflects_to_l1` — ExitingL2 → L1Resumed with vmcb12 populated.
  - `liveness_l2_eventually_exits` — L2 eventually vmexits.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SvmNested::vmrun` post: vcpu.guest_mode == true; vmcb02 active; hsave populated | `SvmNested::vmrun` |
| `SvmNested::vmexit` post: vcpu.guest_mode == false; vmcb01 active; vmcb12 populated with exit-state | `SvmNested::vmexit` |
| `SvmNested::recalc_intercepts` post: vmcb02.intercepts ⊇ vmcb01.intercepts ∪ vmcb12.intercepts | `SvmNested::recalc_intercepts` |
| Per-L2 nested_cr3 validates pointing to valid memslot range | `SvmNested::load_cr3` |
| Per-L1-vCPU at most one vmcb02 active | invariants on enter/exit guest mode |

### Layer 4: Verus/Creusot functional

`L1 VMRUN(vmcb12_gpa) → L2 execution → L2 vmexit → L1 sees vmcb12 populated with exit-info` round-trip equivalence: per-L2-vmexit the L1's vmcb12 contents match what L2 hardware actually saved (modulo emulation).

## Hardening

(Inherits row-1 features from `virt/kvm/x86-svm.md` § Hardening.)

nested-SVM-specific reinforcement:

- **Per-VMRUN vmcb12 validation** — defense against L1-injected invalid state crashing CPU on real-VMRUN.
- **L0 OR-merge intercepts** — defense against L1 disabling intercepts L0 needs.
- **mandatory_exceptions** for #DB / others — defense against L0 losing visibility.
- **Per-L2 ASID distinct** — defense against TLB-leak across L2 instances.
- **Nested-NPT walk-depth bounded** — defense against unbounded TDP walk on pathological L1-NPT.
- **Per-L1-vCPU vmcb02 distinct** — defense against vmcb02 reuse across L1 vCPUs causing state corruption.
- **Per-VMRUN hsave snapshot under preempt-disable** — defense against partial-save on host-IRQ.
- **vmcb12.host-state validated** — defense against L1 setting malicious values escape on L2-vmexit.
- **Per-L2 NPT root invalidate on L2-shutdown** — defense against TDP-stale-mapping for next L2 instance.
- **Per-cycle vmcb02 cache invalidate** before reuse — defense against per-CPU VMCB-cache stale referencing freed memory.
- **AVIC-with-nested validate** — defense against AVIC mis-direction crossing L1/L2 boundary.
- **MSR pass-through bitmap intersect** — defense against L1 leaking direct-MSR-access to L2 for protected MSRs.
- **Per-VM nested-cap gating** via KVM_CAP_NESTED — defense against unauthorized nested-virt enablement.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Nested-VMX (covered in `x86-nested-vmx.md` Tier-3)
- SVM core (covered in `x86-svm.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- SEV/SEV-ES with nested (covered separately in `x86-sev.md` future Tier-3)
- TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- AVIC (covered in `x86-svm.md` Tier-3)
- Implementation code
