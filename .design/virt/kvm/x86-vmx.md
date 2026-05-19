# Tier-3: arch/x86/kvm/vmx/vmx.c — Intel VMX vendor implementation (vmenter/vmexit + per-vCPU VMCS)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/vmx/vmx.c
  - arch/x86/kvm/vmx/vmx.h
  - arch/x86/kvm/vmx/vmcs.h
  - arch/x86/kvm/vmx/vmcs12.c
  - arch/x86/kvm/vmx/vmcs12.h
  - arch/x86/kvm/vmx/vmcs_shadow_fields.h
  - arch/x86/kvm/vmx/main.c
  - arch/x86/kvm/vmx/run_flags.h
  - arch/x86/kvm/vmx/vmenter.S
  - arch/x86/kvm/vmx/capabilities.h
  - arch/x86/kvm/vmx/common.h
-->

## Summary

vmx.c is the Intel-VMX vendor-half of KVM: per-VM/vCPU VMCS allocation + setup, per-feature MSR auto-load lists, vmexit-reason dispatch, posted-interrupt + ept-violation + ept-misconfig + #PF + EPT-violation handlers, and the vmenter assembly stub (vmenter.S) that runs guest code. All non-arch-agnostic VM ops dispatch from kvm_x86_ops to per-vendor function table (kvm_vmx_x86_ops); main.c exports the `vmx_init` module entry that registers vendor ops with KVM core. Single largest VMX-specific file at ~8866 lines.

This Tier-3 covers `arch/x86/kvm/vmx/vmx.c` (~8866) + `vmx.h` (~730) + main.c (~1079) + vmcs.h + vmcs12.{c,h} + vmcs_shadow_fields.h + run_flags.h + vmenter.S + capabilities.h + common.h.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vcpu_vmx` | per-vCPU VMX state | `kernel::kvm::x86::vmx::VcpuVmx` |
| `struct kvm_vmx` | per-VM VMX state | `KvmVmx` |
| `struct vmcs` | VMCS region (host-managed) | `Vmcs` |
| `struct vmcs_host_state` | per-host CPU VMCS state | `VmcsHostState` |
| `struct loaded_vmcs` | per-vCPU VMCS load-state | `LoadedVmcs` |
| `vmx_create_vcpu(vcpu)` | per-vCPU VMX init | `VcpuVmx::create` |
| `vmx_free_vcpu(vcpu)` | per-vCPU VMX teardown | `VcpuVmx::free` |
| `vmx_vcpu_load(vcpu, cpu)` / `vmx_vcpu_put(vcpu)` | per-cpu VMCS load/unload | `VcpuVmx::load` / `_put` |
| `vmx_vcpu_run(vcpu, run_flags)` | top-level run-loop entry | `VcpuVmx::run` |
| `__vmx_vcpu_run(vcpu, regs, run_flags)` (vmenter.S) | asm vmenter stub | `VcpuVmx::__run` |
| `vmx_handle_exit(vcpu, fastpath)` | vmexit-reason dispatch | `VcpuVmx::handle_exit` |
| `vmx_handle_external_interrupt(vcpu)` | EXT_INTERRUPT exit handler | `VcpuVmx::handle_external_intr` |
| `vmx_handle_exception_nmi(vcpu)` | EXCEPTION_NMI exit handler | `VcpuVmx::handle_exception_nmi` |
| `handle_ept_violation(vcpu)` | EPT_VIOLATION exit handler | `VcpuVmx::handle_ept_violation` |
| `handle_ept_misconfig(vcpu)` | EPT_MISCONFIG exit handler | `VcpuVmx::handle_ept_misconfig` |
| `handle_io(vcpu)` / `handle_cr` / `handle_dr` | per-instruction exit handlers | `VcpuVmx::handle_io` / `_cr` / `_dr` |
| `handle_invlpg` / `handle_rdmsr` / `handle_wrmsr` / `handle_cpuid` | exit handlers | per-handler |
| `vmx_set_msr_bitmap(...)` | per-vCPU MSR-bitmap mod | `VcpuVmx::set_msr_bitmap` |
| `vmx_inject_irq(vcpu, ...)` / `vmx_inject_nmi(vcpu)` / `vmx_queue_exception(vcpu, ...)` | event injection | `VcpuVmx::inject_irq` / `_nmi` / `_queue_exception` |
| `vmx_set_cr0` / `set_cr3` / `set_cr4` / `set_efer` | guest reg writes that may VMexit-on-write | per-setter |
| `vmx_set_segment(...)` / `_get_segment(...)` | per-segment R/W | `VcpuVmx::set_segment` / `_get_segment` |
| `setup_vmcs_config(...)` | per-CPU VMX cap detection at init | `VmxBoot::setup_config` |
| `alloc_vmcs(name)` / `free_vmcs(vmcs)` | VMCS region alloc | `Vmcs::alloc` / `_free` |
| `setup_msrs(vmx)` | per-vCPU MSR auto-load/store list | `VcpuVmx::setup_msrs` |
| `vmx_init()` (main.c) | module init: register kvm_x86_ops | `VmxBoot::init` |
| `vmx_exit()` | module exit | `VmxBoot::exit` |
| `kvm_vmx_exit_handlers[]` | per-exit-reason dispatch table | `VcpuVmx::EXIT_HANDLERS` |
| `nested_vmx_*` | nested-VMX surface (delegated) | covered in nested-vmx Tier-3 |

## Compatibility contract

REQ-1: Per-vCPU VMX state (`vcpu_vmx`):
- `loaded_vmcs` (current VMCS struct: vmcs region, host_state, vmcs_host_state).
- `vmcs01` (top-level VMCS for L1 vCPU).
- `nested` (nested-VMX state for L2; if nested-supported).
- `host_state` (per-host-CPU saved state for switch-on-vmexit).
- `msr_bitmap_l01_ipa` (per-vCPU MSR-bitmap pointer).
- `posted_intr_desc_addr` (PI descriptor for posted-IRQ; per-vCPU).
- `regs_avail` (lazy-load tracking of guest GPR fields in vmcs).
- `regs_dirty` (lazy-store tracking).

REQ-2: Per-VM VMX state (`kvm_vmx`):
- `tss_addr` (KVM-managed task-state-segment for guest fallback).
- `apicv_enabled` / `apicv_inhibit_reasons` (APIC-virtualization gates).
- `pi_pending_on` (per-VM posted-IRQ pending list head).
- `nested` (per-VM nested-VMX caps).

REQ-3: VMCS region (`struct vmcs`):
- 4KiB page-aligned.
- First 32-bits = revision-id (CPUID-derived); next 32 = abort-indicator; rest = VMCS data.
- Owned per-vCPU; loaded onto host CPU via VMPTRLD; cleared via VMCLEAR.

REQ-4: vmenter.S (assembly):
- Saves host GPRs to per-CPU VmcsHostState.
- Loads guest GPRs from vcpu.regs into CPU regs.
- Executes VMRESUME (or VMLAUNCH on first-entry).
- On VMexit returns: saves guest GPRs back to vcpu.regs; restores host GPRs; returns vmexit-reason.

REQ-5: Per-exit dispatch (`vmx_handle_exit`):
- Read vmcs.exit_reason.
- Lookup `EXIT_HANDLERS[exit_reason]`.
- Dispatch to handler.
- On handler return ≥ 0: continue to userspace if reason demands, else re-enter.
- Handler return < 0: error to userspace.

REQ-6: EPT-violation handler:
- Read exit qualification (R/W/X access type, GPA).
- Map GPA → user HVA via memslot.
- Page-fault into host-mm via `kvm_handle_page_fault(...)` → `kvm_mmu_page_fault`.
- TDP MMU installs SPTE (or shadow MMU shadow-walks).

REQ-7: I/O exits (`handle_io`):
- Decode I/O port + size + direction from exit qual.
- If port in `vcpu.arch.pio.in_emul`: emulate via in-kernel device (LAPIC, ioapic, pic, pit).
- Else: return to userspace with KVM_EXIT_IO; userspace QEMU emulates.

REQ-8: MSR bitmap:
- Per-vCPU 4KiB bitmap; bit-per-MSR for read + write.
- If bit clear: guest can directly RDMSR/WRMSR without vmexit.
- If bit set: vmexit + KVM emulates.
- `vmx_set_msr_bitmap(vcpu, msr, type, value)` flips bit per-vCPU.

REQ-9: Event injection:
- VM-entry interruption-information field (32-bit) configures injected event.
- `vmx_inject_irq(vcpu, irq)`: set type=external-interrupt + vector=irq.
- `vmx_inject_nmi(vcpu)`: set type=nmi.
- `vmx_queue_exception(vcpu, vector, has_error_code, error_code)`: set type=hw-exception.

REQ-10: Posted-IRQ integration:
- Per-vCPU PostedInterruptDescriptor (PID) at fixed phys-addr.
- vmx_sync_pir_to_irr (called pre-vmenter): scan PIR (PI-bitmap) for set bits; OR into IRR; clear PIR.
- IOMMU IRTE.posted=1 + PDA → IOMMU writes to PIR directly; LAPIC delivers Notification-Vector → vCPU exits or absorbs.

REQ-11: Per-vCPU VMCS lifecycle:
- VMCLEAR on VM-load (clears stale host-cache).
- VMPTRLD on per-CPU vmcs_load (sets current VMCS for VMENTER).
- VMWRITE / VMREAD per-field for setup + readback.
- VMRESUME for second-entry; VMLAUNCH for first.

REQ-12: Per-vCPU MSR auto-load/store list:
- `setup_msrs(vmx)` populates VM-Entry MSR-Load + VM-Exit MSR-Store + VM-Exit MSR-Load lists.
- Auto-loaded MSRs: IA32_EFER, IA32_PERF_GLOBAL_CTRL, IA32_FS_BASE, IA32_GS_BASE.

REQ-13: TDX support gate (tdx.c separate compile-unit):
- If host CPU is TDX-capable + opted-in: VMX runs as TDX-host; guests are TDX VMs (covered separately).

REQ-14: SGX nested-virt support (sgx.c separate):
- Guest SGX support via virtualized ENCLS instruction; covered in vmx-sgx Tier-3 (future).

## Acceptance Criteria

- [ ] AC-1: KVM-VMX module loads on Intel host with VMX-cap CPU: `dmesg | grep "kvm-intel"` shows vendor init.
- [ ] AC-2: Boot Linux guest under qemu+kvm: VM starts; vCPU loops via VMRESUME; vmexits handled.
- [ ] AC-3: 16-vCPU guest stress: 16-vCPU 16GB Linux running stress-ng for 1h without crash.
- [ ] AC-4: APIC-v test: AVIC-enabled guest sees direct IPI delivery without VM-exit; reduced exits via apic_v counter in /proc/interrupts.
- [ ] AC-5: EPT-violation throughput: 8GB guest with 1GB page-table-walk stress; EPT_VIOLATION exits resolve via TDP MMU.
- [ ] AC-6: MSR-bitmap pass-through: guest RDTSC + RDMSR(IA32_TSC_AUX) without vmexit; counter in /sys/kernel/debug/kvm/.
- [ ] AC-7: Posted-IRQ + SR-IOV NIC: passthrough VF; per-VF MSI-X delivered via posted-IRQ; VM-exit count ↓ vs non-posted.
- [ ] AC-8: kvm-unit-tests `vmx` test passes (basic-vmexit, ept, vmcs-shadow).
- [ ] AC-9: Live migration: VMCS state checkpoint + restore via KVM_GET_VCPU_EVENTS / KVM_SET_VCPU_EVENTS roundtrips correctly.

## Architecture

`VmxBoot` is the module-init singleton:

```
struct VmxBoot {
  vmcs_config: VmcsConfig,                  // per-CPU VMX caps detected at init
  vmx_capability: VmxCapability,
  enable_apicv: bool,
  enable_ept: bool,
  enable_unrestricted_guest: bool,
  enable_pml: bool,
}
```

`VcpuVmx` is the per-vCPU struct (one per `Vcpu`):

```
struct VcpuVmx {
  vcpu: KArc<Vcpu>,                         // back-reference
  vmcs01: KBox<LoadedVmcs>,
  loaded_vmcs: KBox<LoadedVmcs>,           // currently-loaded; either vmcs01 or nested.vmcs02
  msr_autoload: MsrAutoload,
  msr_bitmap: KBox<[u8; 4096]>,
  exit_qualification: u64,
  exit_intr_info: u32,
  posted_intr_desc: KBox<PostedIntrDesc>,
  pi_desc_addr: u64,
  regs_avail: u32,                          // bitmap of GPR-cached fields
  regs_dirty: u32,                          // bitmap of GPR-need-store
  guest_msrs: KVec<SharedMsrEntry>,
  host_state: VmcsHostState,
  nested: Option<KBox<VmxNested>>,
}

struct LoadedVmcs {
  vmcs: KBox<Vmcs>,
  msr_bitmap: KBox<[u8; 4096]>,
  shadow_vmcs: Option<KBox<Vmcs>>,         // for vmcs-shadow (nested-vmx accel)
  cpu: AtomicI32,                           // per-CPU last-loaded
  launched: AtomicBool,
  hv_timer_armed: bool,
  ...
}
```

`VcpuVmx::run` flow per iteration:
1. `kvm_arch_vcpu_load`: VMPTRLD vmcs01 if needed.
2. Pre-vmenter: sync_pir_to_irr; load CR0/CR3/CR4 if dirty; load segment regs; restore guest XSAVE.
3. `__run` (vmenter.S):
   - VMRESUME (or VMLAUNCH if first).
   - On vmexit: saved guest GPRs to vcpu.regs.
4. Post-vmexit: read exit_reason; check immediate-fastpath handlers (IPI, MSR_WRITE):
   - If fastpath: handle inline, return CONTINUE.
   - Else: `handle_exit(vcpu, fastpath)`.
5. `handle_exit`: dispatch via EXIT_HANDLERS[reason].
6. Return: NONE → re-enter; need-userspace → return to caller; error → return -EFAULT.

`VcpuVmx::handle_ept_violation`:
1. Read exit_qualification (R/W/X bits, addr-translation bits).
2. Read GPA from vmcs (GUEST_PHYSICAL_ADDR field).
3. Build error_code (PFERR_WRITE_MASK / _USER_MASK / _PRESENT_MASK from exit_qual).
4. `kvm_mmu_page_fault(vcpu, gpa, error_code, NULL, 0)`.
5. If success: re-enter.
6. If userspace required: return.

`VcpuVmx::inject_irq(vcpu, irq, soft)` flow:
1. Compose VM_ENTRY_INTR_INFO_FIELD = (vector | type<<8 | error_code_valid<<11 | valid<<31).
2. VMWRITE to VMCS.
3. On next vmenter: CPU injects event.

`VmxBoot::init` (main.c):
1. Verify CPU has VMX (CPUID feature MSR).
2. Allocate per-CPU VMCS region (revision-id from VMX_BASIC MSR).
3. Per-CPU enable VMX via VMXON.
4. Detect VMX caps: secondary EXEC ctrls, EPT, VPID, etc.
5. Register `vmx_init_ops` with KVM core via `kvm_x86_vendor_init(&vmx_init_ops)`.
6. Module loaded.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vmcs_no_oob` | OOB | per-VMCS read/write via VMWRITE/VMREAD per-field; field-encoding bounded. |
| `loaded_vmcs_per_cpu_unique` | INVARIANT | per-CPU at most one LoadedVmcs in active state; defense against stale-VMCS-on-cpu causing wrong-vCPU-execution. |
| `regs_dirty_implies_writeback` | INVARIANT | every regs_dirty bit eventually causes VMWRITE pre-vmenter. |
| `vmcs_alloc_pool_no_uaf` | UAF | per-vCPU VMCS allocated via KBox; freed at vcpu teardown after VMCLEAR. |

### Layer 2: TLA+

`virt/kvm/vmx_run_loop.tla` (refines vcpu_runloop.tla from earlier Tier-3) models VMX-specific run: state ∈ {NotLoaded, Loaded, Entering, Running, Exiting, Handling, Faulting}; transitions per vmcs_load + VMRESUME + vmexit + handle_exit.

Properties:
- `safety_no_running_without_loaded` — Running implies vmcs vmptrld'ed on this CPU.
- `safety_no_handling_during_running` — handle_exit only after Exiting transition.
- `liveness_handle_terminates` — handle_exit eventually returns Continue/UserspaceReturn/Error.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuVmx::create` post: vmcs01 allocated; msr_bitmap initialized; loaded_vmcs == vmcs01 | `VcpuVmx::create` |
| `VcpuVmx::run` post: vcpu.regs reflect post-vmexit guest state | `VcpuVmx::run` |
| `VcpuVmx::handle_ept_violation` post: SPTE installed iff page available; vmx_continue OR userspace-return | `VcpuVmx::handle_ept_violation` |
| Per-VCPU vmcs01.cpu equals current CPU at vmenter time | `VcpuVmx::vcpu_load` |
| `vmx_inject_irq` post: VM_ENTRY_INTR_INFO_FIELD has valid bit set + correct vector | `VcpuVmx::inject_irq` |

### Layer 4: Verus/Creusot functional

`VcpuVmx::run(vcpu) loop body → guest instruction execution semantically equivalent to per-instruction emulation`: every guest instruction either (a) VMexits to KVM handler that produces same effect as emulator, or (b) executes natively via VMRESUME with VMCS-controlled side-effects.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

VMX-specific reinforcement:

- **VMCLEAR on per-vCPU teardown** + **VMCLEAR on cross-CPU migrate** — defense against stale per-CPU VMCS cache referencing freed memory.
- **Per-CPU VMXON region in IOMMU-protected memory** — defense against device-DMA writing to VMXON region.
- **MSR-bitmap initialized restrictive** (all-set = vmexit-on-access) → only allow direct-access for explicitly-listed MSRs — defense against MSR feature-bypass.
- **EPT-violation handler validates GPA in memslot** — defense against guest-controlled GPA pointing outside guest-owned memory.
- **Posted-IRQ descriptor in IOMMU-mapped + cacheline-aligned** — defense against IOMMU + CPU racing on PIR cacheline.
- **VMX feature-MSR enforcement** at `setup_vmcs_config` — defense against running with unsupported VMX feature combo.
- **vmcs.revision_id matches CPU** — defense against cross-CPU-revision VMPTRLD causing #UD.
- **Per-vCPU regs_avail/regs_dirty discipline** — defense against torn GPR view in nested handlers.
- **EXIT_HANDLERS table bounded** — defense against malformed exit_reason vectoring outside table.
- **Nested-VMX controls validated against vmcs_config** — defense against L2 escape via unsupported control bit.
- **Per-vCPU msr_bitmap distinct from msr_bitmap_legacy** when nested — defense against L1 leaking MSR access to L2.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — VMCS regions kernel-allocated; KVM_GET/SET_NESTED_STATE bounded by `KVM_STATE_NESTED_VMX_VMCS_SIZE`; no raw user copy into VMCS regions.
- **PAX_KERNEXEC** — `vmx_vcpu_run`, `vmx_handle_exit`, `vmx_handle_external_intr`, vendor ops resolve through RX-only kernel text; `kvm_x86_ops` VMX slot table RO.
- **PAX_RANDKSTACK** — VM-entry / VM-exit handler dispatch inherits RANDKSTACK from KVM_RUN.
- **PAX_REFCOUNT** — per-vCPU VMCS, msr_bitmap, vmcs02 nested-cache refcount saturating; concurrent destroy + VMRUN races cannot wrap.
- **PAX_MEMORY_SANITIZE** — per-vCPU VMCS, msr_bitmap, io_bitmap, posted-int desc zeroed on alloc/free; vmcs02 nested-cache zeroed before slab handback so L2 register state never leaks.
- **PAX_UDEREF** — KVM_SET_NESTED_STATE copy_from_user STAC/CLAC bracketed; vmcs12 GPA validated before VMPTRLD-emulation.
- **PAX_RAP / kCFI** — `kvm_x86_ops` VMX table RAP-signed; `kvm_vmx_exit_handlers[]` and per-vmexit handler indirect calls CFI-guarded.
- **GRKERNSEC_HIDESYM** — VMCS phys-addr, msr_bitmap kaddr, posted-int desc kaddr redacted unless CAP_SYSLOG + gr-rbac.
- **GRKERNSEC_DMESG** — VM-entry-failure, VMX-feature-mismatch, vmptrld-mismatch warnings rate-limited and gated by `dmesg_restrict`.
- **KVM ioctl CAP_SYS_ADMIN strict** — KVM_CREATE_VM (VMX), KVM_SET_NESTED_STATE, KVM_CAP_NESTED_STATE gated to CAP_SYS_ADMIN + gr-rbac VM-create role.
- **VMCS PAX_USERCOPY** — VMCS revision_id matches CPU per `setup_vmcs_config`; cross-CPU-revision VMPTRLD rejected to defeat #UD-bait.
- **VMX feature-MSR enforcement** — `vmcs_config` enforced against host VMX capability MSRs (IA32_VMX_BASIC, IA32_VMX_PROCBASED_CTLS, etc); unsupported feature combos rejected at boot.
- **MSR-bitmap host-managed** — `vmx->msr_bitmap` writes mediated by `vmx_disable_intercept_for_msr` / `vmx_enable_intercept_for_msr` under vCPU lock; guest cannot toggle interception directly.
- **EXIT_HANDLERS table bounded** — exit-reason validated against `kvm_vmx_max_exit_handlers`; OOB rejected as KVM_EXIT_INTERNAL_ERROR.
- **Nested-VMX controls validated** — VMCS12 controls validated against host `vmcs_config` before L1→L2 transition; L2 escape via unsupported control bit defended.

Per-doc rationale: VMX is one of two hardware virtualization backends; the grsec reinforcement keeps VMCS authoritative kernel-side, validates VMCS revision_id and feature-MSR consistency, RAP-signs the exit-handler dispatch, and SANITIZEs per-vCPU VMCS/bitmap state so L2 register residue cannot bleed to L1 across nested transitions.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Nested-VMX (covered in `virt/kvm/x86-nested-vmx.md` future Tier-3)
- TDX (covered in `virt/kvm/x86-tdx.md` future Tier-3)
- VMX-SGX nested (covered in `virt/kvm/x86-vmx-sgx.md` future Tier-3)
- Hyper-V VMX accel (hyperv.c covered in `virt/kvm/x86-hyperv.md` future Tier-3)
- AMD SVM (different vendor; covered in `x86-svm.md` future Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- Implementation code
