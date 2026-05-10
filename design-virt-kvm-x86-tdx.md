---
title: "Tier-3: arch/x86/kvm/vmx/tdx.c — Intel TDX (Trust Domain Extensions) confidential VM (TD-vCPU + TDH SEAMCALL + per-TD encrypted memory)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Intel TDX (Trust Domain Extensions) is Intel's confidential-VM equivalent to AMD SEV: a Trust Domain (TD) is a VM whose memory is encrypted via per-TD HKID (Host Key Identifier) + integrity-protected via per-page MAC, isolated from VMM via a trusted "TDX module" running in SEAM (Secure Arbitration Mode). KVM creates TDs via TDH SEAMCALL hypercalls (TDH.MNG.CREATE / TDH.VP.CREATE / TDH.MEM.PAGE.ADD / TDH.MNG.RD / TDH.SYS.INFO / TDH.VP.WR / TDH.SYS.LP.INIT / etc.) handled by the TDX module. Per-TD measurement (MRTD) attestable; per-VP guest registers in encrypted "TDVPS" pages opaque to KVM. Per-TD vmexits surface as TDX-host TDEXIT events with limited info via TDVPS (similar to SEV-ES GHCB).

This Tier-3 covers `arch/x86/kvm/vmx/tdx.c` (~3428 lines).

### Acceptance Criteria

- [ ] AC-1: TDX module loaded on Sapphire Rapids+ host; tdx_hardware_setup succeeds.
- [ ] AC-2: KVM_TDX_INIT_VM: TDR + TDCS allocated; HKID assigned.
- [ ] AC-3: KVM_TDX_INIT_MEM_REGION: 100MB encrypted + measured; SEAMCALL counter increments.
- [ ] AC-4: KVM_TDX_FINALIZE_VM: measurement (MRTD) extracted + matches expected.
- [ ] AC-5: TD VM boots: 4-vCPU TD running Linux; vcpu_run loop active.
- [ ] AC-6: TDVMCALL: TD MMIO write trapped; KVM emulates; result returned via TDVPS.
- [ ] AC-7: TD vmexit: TD #PF resolved via S-EPT page-add; TD resumes.
- [ ] AC-8: Posted-IRQ to TD: KVM IPI to TD vCPU; TD sees IRQ; no encrypted-state leak.
- [ ] AC-9: TD teardown: TDH.MEM.PAGE.REMOVE per-page + TDH.MNG.RESID; HKID released.
- [ ] AC-10: kvm-unit-tests `tdx` test passes (where supported).

### Architecture

`KvmTdx` per-VM:

```
struct KvmTdx {
  state: AtomicI32,                          // TDX_VM_STATE_*
  tdr_pa: u64,
  tdcs_pa: KVec<u64>,                         // typically 8 pages
  hkid: u32,
  attributes: u64,
  xfam: u64,
  tsc_offset: u64,
  ...
}

struct VcpuTdx {
  tdvpr_pa: u64,
  tdvps_pa: KVec<u64>,                        // typically 6 pages
  vp_index: u32,
  pi_desc: KBox<PostedIntrDesc>,
  pi_desc_addr: u64,
  guest_state_protected: bool,
  interrupt_disabled_from_root_seam: bool,
  ...
}
```

`KvmTdx::vm_init(kvm)` (KVM_VM_TYPE_TDX):
1. Allocate kvm.arch.kvm_tdx.
2. Acquire HKID from pool.
3. SEAMCALL TDH.MNG.CREATE → tdr_pa.
4. SEAMCALL TDH.MNG.ADDCX (multiple times) → tdcs_pa array.
5. State = INITIALIZED.

`VcpuTdx::create(vcpu)`:
1. Allocate kvm.arch.vcpu_tdx.
2. SEAMCALL TDH.VP.CREATE → tdvpr_pa.
3. SEAMCALL TDH.VP.ADDCX (multiple times) → tdvps_pa array.
4. SEAMCALL TDH.VP.INIT → per-VP setup with initial RIP / GPRs.

`VcpuTdx::run(vcpu)` flow:
1. Pre-run: any pending TDVPS writes via TDH.VP.WR.
2. SEAMCALL TDH.VP.ENTER (vcpu.tdvpr_pa) — actual TD execution.
3. On return: parse exit-info from registers.
4. tdx_handle_exit(vcpu):
   - exit_reason ∈ {TDX_EXIT_REASON_*}.
   - TDX_EXIT_REASON_TDVMCALL: parse TDVMCALL sub-function.
   - TDX_EXIT_REASON_EPT_VIOLATION: S-EPT-page-add via TDH.MEM.PAGE.AUG.
   - TDX_EXIT_REASON_INTERRUPT: external IRQ; resume.
5. Re-enter on next loop iteration.

`Tdx::seamcall(fn, args, leaf)`:
1. Set RAX = fn (SEAMCALL leaf).
2. Set RCX/RDX/R8/R9/R10 = args.
3. Execute SEAMCALL instruction → dispatches to TDX module in SEAM.
4. On return: read RAX (status), other regs (output).

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- TDX module driver (arch/x86/virt/vmx/tdx; covered in `arch/x86/virt-tdx.md` future Tier-3)
- TDX firmware loading (firmware concern)
- TD migration (separate; covered in `x86-tdx-migration.md` future Tier-3)
- AMD SEV (covered in `x86-sev.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_tdx` | per-VM TDX state | `kernel::kvm::x86::vmx::tdx::KvmTdx` |
| `struct vcpu_tdx` | per-vCPU TDX state | `VcpuTdx` |
| `struct tdx_module_args` | TDX module SEAMCALL args | `TdxModuleArgs` |
| `tdx_hardware_setup()` | per-host TDX init at module load | `Tdx::hardware_setup` |
| `tdx_hardware_teardown()` | per-host teardown | `Tdx::hardware_teardown` |
| `tdx_offline_cpu()` | per-CPU offline hook | `Tdx::offline_cpu` |
| `tdx_init()` | TDX-module init | `Tdx::init` |
| `tdx_vm_init(kvm)` (KVM_VM_TYPE_TDX) | per-VM init | `KvmTdx::vm_init` |
| `tdx_vm_destroy(kvm)` | per-VM teardown | `KvmTdx::vm_destroy` |
| `tdx_vcpu_create(vcpu)` | per-vCPU init | `VcpuTdx::create` |
| `tdx_vcpu_free(vcpu)` | per-vCPU teardown | `VcpuTdx::free` |
| `tdx_vcpu_run(vcpu)` | TDX vmenter+vmexit (TDH.VP.ENTER) | `VcpuTdx::run` |
| `tdx_handle_exit(vcpu, fastpath)` | TDX vmexit handler | `VcpuTdx::handle_exit` |
| `tdh_mng_create(...)` | TDH.MNG.CREATE | `TdxSeamcall::mng_create` |
| `tdh_vp_create(...)` | TDH.VP.CREATE | `TdxSeamcall::vp_create` |
| `tdh_mem_page_add(...)` | TDH.MEM.PAGE.ADD (encrypt + measure page) | `TdxSeamcall::mem_page_add` |
| `tdh_mem_sept_add(...)` | TDH.MEM.SEPT.ADD (Secure EPT) | `TdxSeamcall::mem_sept_add` |
| `tdh_mng_rd(...)` | TDH.MNG.RD (read TDCS) | `TdxSeamcall::mng_rd` |
| `tdh_mng_finalize(...)` | TDH.MNG.FINALIZE (lock TD measurement) | `TdxSeamcall::mng_finalize` |
| `tdh_vp_init(...)` | TDH.VP.INIT | `TdxSeamcall::vp_init` |
| `tdh_vp_rd(...)` | TDH.VP.RD (read TDVPS field) | `TdxSeamcall::vp_rd` |
| `tdh_vp_wr(...)` | TDH.VP.WR (write TDVPS field) | `TdxSeamcall::vp_wr` |
| `seamcall(fn, args, leaf)` | low-level SEAMCALL dispatcher | `Tdx::seamcall` |
| `tdvmcall_*` (set of guest TDVMCALLs) | guest-issued vmcall dispatch | `Tdx::tdvmcall` |
| `tdx_handle_dr7_write(...)` | DR7 write intercept | `VcpuTdx::handle_dr7_write` |

### compatibility contract

REQ-1: Per-VM `kvm_tdx`:
- `state` (TDX_VM_STATE_NOT_INITIALIZED / _INITIALIZED / _CONFIGURED / _RUNNING / _TEARDOWN).
- `tdr_pa` (TDR = Trust Domain Root page; physical addr).
- `tdcs_pa` (TDCS = Trust Domain Control Structure pages; array).
- `hkid` (per-VM Host Key ID; assigned from TDX-pool).
- `attributes` (per-TD attributes: TUD = Trust-Domain-User-Definable bits).
- `xfam` (per-TD XCR0 mask).
- `tsc_offset` (per-TD TSC offset).
- `cpuid_*` (per-TD CPUID configuration).
- `migration_state` (for TD migration).

REQ-2: Per-vCPU `vcpu_tdx`:
- `tdvpr_pa` (TDVPR = TD-VP-Root page).
- `tdvps_pa` (TDVPS = TD-VP-State pages array; encrypted vCPU state).
- `vp_index` (per-VP index in TD).
- `interrupt_disabled_from_root_seam` (KVM tracking).
- `pi_desc` (posted-IRQ descriptor; shared with TDX module).
- `guest_state_protected` (always true once finalized).

REQ-3: TD lifecycle states:
- NOT_INITIALIZED: KVM_VM_TYPE_TDX created; awaiting TDH.MNG.CREATE.
- INITIALIZED: TDR allocated + per-CPU TDCS extended.
- CONFIGURED: TDH.MNG.INIT done; per-VP set up.
- RUNNING: TDH.MNG.FINALIZE called; measurement locked; TD running.
- TEARDOWN: per-page TDH.MEM.PAGE.REMOVE + TDH.MNG.VPFLUSHDONE + TDH.MNG.RESID.

REQ-4: TDX hardware setup (`tdx_hardware_setup`):
1. Verify CPU has TDX (CPUID + IA32_VMX_BASIC).
2. Per-CPU SEAMCALL TDH.SYS.LP.INIT (initialize TDX module on this LP).
3. Allocate HKID pool (hardware-managed range).
4. KVM advertises KVM_VM_TYPE_TDX.

REQ-5: KVM_TDX_INIT_VM ioctl flow:
1. SEAMCALL TDH.MNG.CREATE → TDR page allocated.
2. SEAMCALL TDH.MNG.ADDCX → multiple TDCS pages added.
3. SEAMCALL TDH.MNG.INIT → TD configured with attributes/CPUID/XFAM.
4. Per-CPU SEAMCALL TDH.MNG.KEYCONFIG → per-CPU keys configured.

REQ-6: KVM_TDX_INIT_MEM_REGION (per-page add):
1. Map source data into kernel.
2. SEAMCALL TDH.MEM.PAGE.ADD → encrypt + measure page; copy to TD's encrypted memory.
3. Repeat per-page.

REQ-7: KVM_TDX_FINALIZE_VM:
1. SEAMCALL TDH.MNG.FINALIZE → lock TD measurement (MRTD); transitions to RUNNING.
2. After this: no more TDH.MEM.PAGE.ADD permitted.

REQ-8: TD vCPU run flow (`tdx_vcpu_run`):
1. Pre-run: configure TDVPS via TDH.VP.WR (e.g., RIP, RSP, GPRs).
2. SEAMCALL TDH.VP.ENTER (the actual TD execution).
3. TDX module performs VMENTER on encrypted state.
4. TD vCPU runs; on vmexit:
   - SEAMCALL returns with TD-EXIT info in registers.
   - Some exits surfaced to KVM (e.g., MMIO emulation needed).
   - Some handled internally by TDX module.

REQ-9: TDVMCALL guest-issued hypercall:
- Guest in TD executes TDVMCALL; surfaces to KVM with limited info.
- Equivalent to GHCB-VMGEXIT in SEV-ES.
- Allow-listed exit_codes only.
- Used for: MMIO read/write, MSR read/write, IO port, instruction-emulation requests.

REQ-10: Secure EPT (S-EPT):
- Per-TD nested page table managed by TDX module (not KVM).
- KVM cannot read S-EPT contents.
- TDH.MEM.PAGE.AUG / TDH.MEM.PAGE.PROMOTE for page additions.
- TDH.MEM.RANGE.BLOCK / TDH.MEM.RANGE.UNBLOCK for permission changes.

REQ-11: Per-TD HKID:
- Hardware Key Identifier; per-TD encryption key (managed by TDX module).
- HKIDs from a global pool; recycle on TD teardown (with explicit TDH.MNG.VPFLUSHDONE flush).

REQ-12: Posted-IRQ for TDX:
- KVM cannot directly inject IRQs (would corrupt encrypted state).
- Per-vCPU posted-IRQ descriptor in shared memory; TDX module observes + injects.
- Limited to specific vector ranges per spec.

REQ-13: TDX module errors via TDX errno (tdx_errno.h):
- Per-SEAMCALL return code; KVM dispatches via central error map.
- Recovery: per-error type, may need page-add retry / vCPU re-init / TD-teardown.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `hkid_unique_per_td` | INVARIANT | per-TD HKID distinct across all running TDs at any time. |
| `tdr_tdcs_aligned` | INVARIANT | per-TD TDR + TDCS pages 4KiB-aligned. |
| `tdvps_aligned` | INVARIANT | per-vCPU TDVPS pages 4KiB-aligned. |
| `state_transitions_valid` | INVARIANT | per-TD state transitions follow NOT_INIT → INIT → CONFIG → RUNNING → TEARDOWN. |
| `seamcall_args_valid` | INVARIANT | per-SEAMCALL args validated against TDX-module-spec; defense against malformed-call crashing host. |

### Layer 2: TLA+

`virt/kvm/tdx_lifecycle.tla`:
- States: NotInitialized, Initialized, Configured, Running, Teardown.
- Properties:
  - `safety_finalize_locks_measurement` — Running implies MRTD frozen.
  - `safety_no_page_add_after_finalize` — TDH.MEM.PAGE.ADD rejected post-finalize.
  - `liveness_init_eventually_running` — assuming valid path, INIT eventually Running.

`virt/kvm/tdx_seamcall_dispatch.tla`:
- Per-SEAMCALL state: Pending, Executing, Returned.
- Properties:
  - `safety_seamcall_atomic` — TDX module sees consistent args; KVM sees consistent return.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `KvmTdx::vm_init` post: TDR allocated; HKID assigned; state == INITIALIZED | `KvmTdx::vm_init` |
| `VcpuTdx::create` post: TDVPR + TDVPS allocated; tdh_vp_init succeeded | `VcpuTdx::create` |
| `VcpuTdx::run` post: per-vmexit info parsed; appropriate handler called | `VcpuTdx::run` |
| Per-TD HKID released on vm_destroy with VPFLUSHDONE | `KvmTdx::vm_destroy` |
| Per-SEAMCALL error code propagated to userspace via ioctl | invariants on SEAMCALL dispatch |

### Layer 4: Verus/Creusot functional

`KVM_TDX_INIT_VM + INIT_MEM_REGION + FINALIZE_VM = TD with attestable measurement equal to hash of measured-loaded-encrypted-firmware`: per-TD the MRTD output represents cryptographic hash of the loaded firmware code as encrypted by TDX module.

### hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

TDX-specific reinforcement:

- **Per-TD HKID distinct + cleared on teardown** — defense against per-TD-key leak across confidential VMs.
- **Per-TD TDR + TDCS in TDX-module-protected memory** — defense against host-direct-write corrupting TD state.
- **SEAMCALL args validated against TDX-module-spec** — defense against malformed-call causing TDX-module-fault.
- **Per-vCPU TDVPS opaque to KVM** — defense against direct register-read leaking confidential state.
- **TDVMCALL exit-code allow-list** — defense against TD reaching unsupported handler paths.
- **Posted-IRQ vector validated** — defense against KVM injecting privileged-vector to TD.
- **MRTD locked at FINALIZE** — defense against post-finalize tampering.
- **Per-CPU TDX-module init via TDH.SYS.LP.INIT** — defense against running TD on uninitialized LP.
- **Page-add only pre-FINALIZE** — defense against post-finalize tampering attempt.
- **TDH.MNG.VPFLUSHDONE before HKID reuse** — defense against stale-key-ID causing cross-TD memory leak.
- **S-EPT managed by TDX-module** — defense against KVM-side page-table tampering.
- **Per-vCPU pi_desc in IOMMU-protected shared memory** — defense against device-DMA writing protected fields.

