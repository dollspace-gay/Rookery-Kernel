# Tier-3: arch/x86/kvm/vmx/sgx.c — SGX-in-VM virtualization (per-vCPU ENCLS/ENCLU emulation + EPC virtualization)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-vmx.md
upstream-paths:
  - arch/x86/kvm/vmx/sgx.c
  - arch/x86/kvm/vmx/sgx.h
  - arch/x86/kvm/vmx/vmx.c (SGX exit handling)
-->

## Summary

Intel SGX (Software Guard Extensions) lets userspace create per-process "enclaves" (isolated memory regions encrypted at RAM controller via per-CPU MEE-key) where code+data is protected from kernel/hypervisor read. KVM emulates SGX-in-VM so guest userspace can use SGX inside a VM: per-VM SGX enabled via CPUID + SGX-LC (Launch Control) MSRs; ENCLS leaf instructions (privileged) intercepted + emulated by KVM if guest issues from kernel-mode (vmexit reason = SGX); ENCLU leaf instructions (user) executed natively in guest (no vmexit — uses guest-mapped EPC pages). Per-VM EPC (Enclave Page Cache) regions allocated from host's EPC; KVM maps into guest EPT.

This Tier-3 covers `arch/x86/kvm/vmx/sgx.c` (~510 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `handle_encls(vcpu)` | ENCLS vmexit handler | `Sgx::handle_encls` |
| `handle_encls_ecreate(vcpu)` | ENCLS[ECREATE] | `Sgx::handle_ecreate` |
| `handle_encls_einit(vcpu)` | ENCLS[EINIT] | `Sgx::handle_einit` |
| `kvm_intel_sgx_vmcs_setup(...)` | per-vCPU VMCS SGX setup | `VcpuVmx::sgx_vmcs_setup` |
| `kvm_vmx_sgx_supported()` | per-host detection | `Sgx::supported` |
| `vmx_get_msr_feature(...)` SGX-LC branches | MSR_IA32_SGXLEPUBKEYHASH* | covered in MSR handler |
| `kvm_intel_sgx_lc_msrs_show(...)` | per-vCPU SGX-LC MSR display | `VcpuVmx::sgx_lc_msrs` |
| `setup_sgx_resources(vcpu)` | per-VM EPC alloc | `Vmx::setup_sgx_resources` |
| `nested_handle_encls(...)` | nested-VMX ENCLS | covered in nested.c |
| `sgx_get_encls_gva(vcpu, leaf, ...)` | guest-VA decode | `Sgx::get_encls_gva` |
| `__handle_encls_ecreate(...)` | actual ECREATE emulation | `Sgx::__handle_ecreate` |

## Compatibility contract

REQ-1: Per-VM SGX state:
- `sgx_provisioned` (bool: SGX-on-VM enabled).
- `sgx_lepubkeyhash[4]` (SGX Launch-Control public-key-hash MSRs; per-VM).
- `sgx_lc_locked` (LC-MSR locked / unlocked).
- `epc_regions` (KVec of EPC sub-regions assigned to this VM).

REQ-2: SGX CPUID enumeration:
- 0x12 sub-leaf 0: SGX1 + SGX2 features.
- 0x12 sub-leaf 1: ATTRIBUTES MSR mask + XCR0 mask.
- 0x12 sub-leaf N>=2: per-EPC-section enumeration.

REQ-3: ENCLS leaf instructions (privileged):
- Subleaves: ECREATE (0x0), EADD (0x1), EINIT (0x2), EREMOVE (0x3), EDBGRD/WR (0x4/0x5), EXTEND (0x6), ELDB/U (0x7/0x8), EBLOCK (0x9), EPA (0xA), EWB (0xB), ETRACK (0xC), EAUG (0xD), EMODPR (0xE), EMODT (0xF).
- Each requires CPL=0 (kernel mode); otherwise #UD.
- ENCLS vmexit when guest in CPL=0 issues; KVM emulates per-leaf.

REQ-4: ENCLU leaf instructions (user):
- Subleaves: EREPORT (0x0), EGETKEY (0x1), EENTER (0x2), ERESUME (0x3), EEXIT (0x4), EACCEPT (0x5), EMODPE (0x6), EACCEPTCOPY (0x7).
- CPL=3 (user mode); execute natively.
- No vmexit by default (guest userspace uses directly).

REQ-5: ENCLS[ECREATE] emulation:
1. Read SECS (Secure Enclave Control Structure) from guest-VA.
2. Validate fields: ATTRIBUTES, MISCSELECT, XFRM (intersect with sgx_lepubkeyhash mask).
3. Forward to host SGX driver via PR_SET_MM-like interface OR perform native ENCLS in host context with per-VM EPC region.

REQ-6: ENCLS[EINIT] emulation:
1. Read SIGSTRUCT (signed enclave structure) from guest-VA.
2. Validate signing-key-hash matches vcpu.arch.sgx_lepubkeyhash.
3. Forward to host EINIT.

REQ-7: SGX-LC MSRs (Launch-Control):
- MSR_IA32_SGXLEPUBKEYHASH0..3 (0x8C..0x8F): 256-bit hash of authorized launch-key.
- Locked via MSR_IA32_FEATURE_CONTROL.lock = 1.
- KVM virtualizes per-VM (override host's hash for guest).

REQ-8: EPC allocation:
- Per-VM allocate sub-region from host EPC (privileged).
- Map into guest EPT at agreed GPA range.
- Per-VM accounting via memcg (sgx-epc cgroup controller).

REQ-9: SGX in nested-VMX:
- L1 may also virtualize SGX for L2.
- Nested ENCLS routed via nested.c.

REQ-10: Per-vCPU XFRM (XSAVE feature mask) for SGX:
- ATTRIBUTES.XFRM controls which XSAVE features available inside enclave.
- KVM intersects vCPU.XCR0-mask with host SGX-supported mask.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest under qemu+kvm with `+sgx`: guest CPUID 0x12 enumerates SGX1/SGX2.
- [ ] AC-2: ENCLS[ECREATE] from guest kernel: vmexit; KVM emulates; SECS allocated.
- [ ] AC-3: ENCLU[EENTER] from guest userspace: native execution; enclave entered.
- [ ] AC-4: SGX-LC: guest writes MSR_IA32_SGXLEPUBKEYHASH; per-VM hash overridden.
- [ ] AC-5: Host EPC partitioning: 4 VMs each with 16MB EPC; per-VM isolated; aggregate ≤ host EPC size.
- [ ] AC-6: Live migration: guest with active SGX enclave migrated; enclave state preserved.
- [ ] AC-7: Spec-violation defense: ENCLS leaf with invalid leaf number raises #GP to guest.
- [ ] AC-8: kvm-unit-tests `sgx` test passes.
- [ ] AC-9: Nested-SGX (L1 KVM with SGX-on-VM, L2 with SGX-on-L1): ENCLS path works.

## Architecture

`KvmSgx` per-VM (in kvm.arch):

```
struct KvmSgx {
  provisioned: bool,
  sgx_lepubkeyhash: [u64; 4],
  lc_locked: bool,
  epc_regions: KVec<KvmSgxEpcRegion>,
}

struct KvmSgxEpcRegion {
  base_gpa: u64,
  size: u64,
  host_pa: u64,                                // physical addr in host EPC
}
```

`Sgx::handle_encls(vcpu)` flow:
1. Read RAX (ENCLS leaf number).
2. Switch:
   - LEAF_ECREATE: Sgx::handle_ecreate(vcpu).
   - LEAF_EADD: Sgx::handle_eadd(vcpu).
   - LEAF_EINIT: Sgx::handle_einit(vcpu).
   - LEAF_EREMOVE: Sgx::handle_eremove(vcpu).
   - default: inject #GP to guest.

`Sgx::handle_einit(vcpu)`:
1. RBX := SIGSTRUCT GVA.
2. RCX := SECS GVA.
3. RDX := EINITTOKEN GVA (or 0 if unsigned).
4. Translate GVA → GPA → host-VA via memslot.
5. Read SIGSTRUCT.
6. Validate sig-hash against vcpu.arch.sgx_lepubkeyhash.
7. If mismatch: set RAX = SGX_INVALID_SIG_STRUCT.
8. Else: forward to host SGX driver via internal API; set RAX = SGX_SUCCESS or per-error.

`VcpuVmx::sgx_vmcs_setup(vcpu)` (called at vCPU init):
1. Set vmcs.SGX_BASE_OFFSET in encls-exiting bitmap (force vmexit on ENCLS).
2. SGX-EPCM-violations not exited (handled by HW per-EPC).
3. Configure ENCLS-EXITING-BITMAP for selected leaves only.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `encls_leaf_validated` | INVARIANT | per-leaf number validated against supported list; defense against guest reaching unimplemented leaf. |
| `epc_region_no_overlap` | INVARIANT | per-VM EPC regions non-overlapping with other VMs'. |
| `sigstruct_hash_validated` | INVARIANT | per-EINIT sig-hash compared with sgx_lepubkeyhash; defense against unauthorized launch. |
| `sgx_lc_locked_immutable` | INVARIANT | sgx_lc_locked transitions only false→true; never unlocked. |

### Layer 2: TLA+

`virt/kvm/sgx_encls_dispatch.tla`:
- Per-vmexit ENCLS leaf state ∈ {Pending, Validating, Emulating, Returned}.
- Properties:
  - `safety_per_leaf_emulated_or_rejected` — every ENCLS leaf either emulated OR injected as #GP.
  - `safety_einit_only_with_valid_sig` — EINIT proceeds only if sigstruct hash matches.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sgx::handle_encls` post: per-leaf return code in RAX; vCPU resumable | `Sgx::handle_encls` |
| `Sgx::handle_einit` post: hash compared; valid → forward to host; invalid → SGX_INVALID_SIG_STRUCT | `Sgx::handle_einit` |
| Per-VM EPC region GPA range bounded; no overlap | `Vmx::setup_sgx_resources` |
| Per-vCPU SGX-LC MSRs persisted across vmexits | `VcpuVmx::sgx_lc_msrs` |

### Layer 4: Verus/Creusot functional

`Guest ENCLS leaf execution → KVM emulation → host SGX driver invocation → SGX hardware operation` semantic equivalence: per-leaf the host-side SGX state change matches the guest's intended operation (modulo error-path differences).

## Hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

SGX-specific reinforcement:

- **Per-leaf number validation** — defense against guest reaching unimplemented ENCLS path.
- **Per-VM EPC region distinct** — defense against cross-VM EPC sharing causing enclave-data leak.
- **sigstruct hash validated against per-VM lepubkeyhash** — defense against unauthorized enclave launch.
- **sgx_lc_locked one-way transition** — defense against guest unlocking after lock.
- **ENCLS-EXITING-BITMAP restrictive** — defense against guest reaching uncontrolled ENCLS leaves.
- **Per-VM EPC accounting via memcg** — defense against single-VM exhausting host EPC.
- **EINITTOKEN validation** — defense against attacker-supplied token bypassing launch-control.
- **Per-VM enclave count bounded** — defense against unbounded enclave-create exhausting EPC.
- **SGX-LC MSRs reset on KVM_CAP_DISABLE_QUIRK** — defense against carryover sigstruct-hash from prior VM lifecycle.
- **SECS XFRM intersect with host XCR0-supported** — defense against guest enabling unsupported XSAVE feature in enclave.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- Host SGX driver (drivers/sgx; covered in `drivers/sgx.md` future Tier-3)
- SGX-LC userspace tools
- Implementation code
