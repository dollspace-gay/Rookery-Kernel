# Tier-3: arch/x86/kvm/svm/svm.c — AMD SVM vendor implementation (vmenter/vmexit + per-vCPU VMCB + AVIC)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/svm/svm.c
  - arch/x86/kvm/svm/svm.h
  - arch/x86/kvm/svm/vmenter.S
  - arch/x86/kvm/svm/avic.c
  - arch/x86/kvm/svm/svm_ops.h
  - arch/x86/kvm/svm/svm_onhyperv.c
-->

## Summary

svm.c is the AMD-SVM vendor-half of KVM: per-vCPU VMCB allocation + setup, AVIC (Advanced Virtual Interrupt Controller) for low-overhead IPI, NPT (Nested Page Tables, AMD's EPT equivalent) integration with TDP MMU, vmexit handler dispatch, and the assembly vmenter stub (vmenter.S) that runs guest code via VMRUN. Mirror of vmx.c on the AMD side, but with VMCB instead of VMCS, simpler host-state save/restore (host-state automatically saved by VMRUN), AVIC instead of APICv, and SEV/SEV-ES/SEV-SNP for confidential VMs.

This Tier-3 covers `svm.c` (~5747) + `svm.h` (~1044) + `vmenter.S` (~404) + `avic.c` (~1319) + `svm_ops.h` + `svm_onhyperv.c` (Hyper-V acceleration helpers).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vcpu_svm` | per-vCPU SVM state | `kernel::kvm::x86::svm::VcpuSvm` |
| `struct kvm_svm` | per-VM SVM state | `KvmSvm` |
| `struct vmcb` | VMCB (Virtual Machine Control Block) | `Vmcb` |
| `struct vmcb_save_area` | VMCB save area (host+guest state) | `VmcbSaveArea` |
| `struct vmcb_control_area` | VMCB control area (intercepts + I/O bmp + MSR bmp) | `VmcbControlArea` |
| `svm_create_vcpu(vcpu)` / `svm_free_vcpu(vcpu)` | per-vCPU init/teardown | `VcpuSvm::create` / `_free` |
| `svm_vcpu_load(vcpu, cpu)` / `svm_vcpu_put(vcpu)` | per-cpu load/put | `VcpuSvm::load` / `_put` |
| `svm_vcpu_run(vcpu, run_flags)` | main run-loop | `VcpuSvm::run` |
| `__svm_vcpu_run(vcpu)` (vmenter.S) | asm VMRUN stub | `VcpuSvm::__run` |
| `handle_exit(vcpu, exit_fastpath)` | exit dispatch | `VcpuSvm::handle_exit` |
| `npf_interception(vcpu)` | NPT-fault handler | `VcpuSvm::handle_npf` |
| `pf_interception(vcpu)` | guest-#PF handler (shadow mode) | `VcpuSvm::handle_pf` |
| `intr_interception(vcpu)` / `nmi_interception(vcpu)` | INTR / NMI exit handlers | `VcpuSvm::handle_intr` / `_nmi` |
| `io_interception(vcpu)` | I/O port exit handler | `VcpuSvm::handle_io` |
| `cpuid_interception(vcpu)` | CPUID exit handler | `VcpuSvm::handle_cpuid` |
| `msr_interception(vcpu)` | RDMSR/WRMSR exit handler | `VcpuSvm::handle_msr` |
| `svm_set_msr_bitmap(...)` | per-vCPU MSR-bitmap mod | `VcpuSvm::set_msr_bitmap` |
| `svm_inject_irq(...)` / `svm_inject_nmi(...)` / `svm_queue_exception(...)` | event injection via VMCB.event_inj | `VcpuSvm::inject_*` |
| `avic_init_vcpu(...)` (avic.c) | per-vCPU AVIC init | `Avic::init_vcpu` |
| `avic_set_running(...)` / `avic_vcpu_load(...)` / `avic_vcpu_put(...)` | AVIC physical-id table mgmt | `Avic::*` |
| `avic_incomplete_ipi_interception(...)` | AVIC IPI fast-path exit | `Avic::handle_incomplete_ipi` |
| `svm_init()` (main entry) | module init | `SvmBoot::init` |
| `svm_set_gif(vcpu, enable)` | global-interrupt-flag toggle | `VcpuSvm::set_gif` |

## Compatibility contract

REQ-1: Per-vCPU SVM state (`vcpu_svm`):
- `vmcb` (current VMCB pointer; either vmcb01 or nested.vmcb02).
- `vmcb01` (top-level VMCB).
- `vmcb_pa` (VMCB physical addr; passed to VMRUN).
- `current_vmcb` (per-CPU last-loaded; for cache flush).
- `nested` (nested-SVM state).
- `host_save` (host save area for VMRUN).
- `nmi_masked` / `nmi_singlestep_guest_rflags`.
- `avic_logical_id_table_page` / `avic_physical_id_cache` (AVIC support).
- `sev_data` (SEV/SEV-ES guest data; covered separately).

REQ-2: VMCB (4KiB):
- Control area (offset 0): intercept bitmaps, IOPM/MSRPM addrs, ASID, TPR, intr ctrl, nested CR3, EVENTINJ, EXITCODE/EXITINFO, NRIP, etc.
- Save area (offset 0x400): per-vCPU complete CPU state (CRn, RIP, RFLAGS, RSP, segments, MSRs).

REQ-3: VMRUN (instruction):
- Saves host state to host_save area.
- Loads guest state from VMCB.save_area.
- Executes guest until vmexit.
- On vmexit: writes guest state back to VMCB.save_area; loads host_save back.
- Atomic from CPU perspective; no host code runs between state save+load.

REQ-4: Per-exit dispatch (`handle_exit`):
- Read VMCB.control.exit_code.
- Lookup `svm_exit_handlers[exit_code]`.
- Dispatch.

REQ-5: NPT (Nested Page Tables):
- AMD's TDP equivalent; per-VM root NPT installed in VMCB.control.nested_cr3.
- NPT-fault → `npf_interception` → `kvm_handle_page_fault` → TDP MMU (covered in x86-mmu-tdp Tier-3).

REQ-6: AVIC (avic.c):
- Per-VM physical-id table (4KiB; one entry per LAPIC).
- Per-VM logical-id table (4KiB).
- Per-VM AVIC backing pages (4KiB-per-vCPU LAPIC-page mapped IO-coherent).
- IPI from guest CPU-A to CPU-B: AVIC HW writes VAPIC backing page of CPU-B + raises doorbell; CPU-B sees IRR change without VM-exit (if AVIC running).
- AVIC-incomplete-IPI exit fired only for non-trivial IPI (IPI to non-running vCPU).

REQ-7: GIF (Global Interrupt Flag):
- Hardware-managed; controls whether guest takes IRQ/NMI/exceptions.
- STGI / CLGI instructions (only valid in SVM mode) toggle.

REQ-8: Per-vCPU intercept bitmaps:
- `vmcb.control.intercept_cr` (bitmap of CR-read/write to intercept).
- `vmcb.control.intercept_dr` (DR-read/write intercept).
- `vmcb.control.intercept_exceptions` (per-vector intercept).
- `vmcb.control.intercept` (general intercept for INTR, NMI, SMI, INIT, VMRUN, etc.).

REQ-9: MSR pass-through bitmap (MSRPM):
- 8KiB bitmap; 2 bits per MSR (read intercept + write intercept).
- Per-vCPU pointer in VMCB.control.msrpm_base_pa.

REQ-10: I/O port pass-through bitmap (IOPM):
- 12KiB bitmap; 1 bit per port × 4 sizes = 8KB but actually 12KB for PMIO range.
- Per-vCPU pointer in VMCB.control.iopm_base_pa.

REQ-11: Event injection:
- `vmcb.control.event_inj` (32-bit field): vector | type<<8 | error_code_valid<<11 | valid<<31.
- VMRUN injects on next entry.
- Type ∈ {INTR, NMI, EXCEPTION, SOFT_INTR}.

REQ-12: Per-VM ASID (Address Space Identifier):
- 32-bit per-VM ID assigned by KVM at VM-create.
- VMCB.control.asid populated.
- Used by AMD IOMMU + TLB to tag per-VM TLB entries (avoids full TLB-flush on VM-switch).

REQ-13: SEV-ES / SEV-SNP support gated:
- SEV (Secure Encrypted Virtualization) encrypts guest memory via per-VM key.
- SEV-ES additionally encrypts vCPU register state.
- SEV-SNP adds attestation + integrity.
- Implementation in svm/sev.c (covered separately).

REQ-14: Hyper-V on SVM acceleration (svm_onhyperv.c):
- When KVM running on Hyper-V hypervisor + Hyper-V-enlightened guests: use Hyper-V-specific MSRs for performance.

## Acceptance Criteria

- [ ] AC-1: KVM-AMD module loads on AMD Zen+ host with SVM-cap CPU: `dmesg | grep "kvm-amd"` shows vendor init.
- [ ] AC-2: Boot Linux guest under qemu+kvm: VM starts; vCPU loops via VMRUN; vmexits dispatched.
- [ ] AC-3: 16-vCPU guest stress: 16-vCPU 16GB Linux running stress-ng for 1h without crash.
- [ ] AC-4: AVIC test: AVIC-enabled guest sees direct IPI delivery without vmexit; counter shows IPI-bypass.
- [ ] AC-5: NPT-fault throughput: 8GB guest with 1GB page-table-walk stress; npf_interception resolves via TDP MMU.
- [ ] AC-6: MSR-pass-through: guest RDTSC + RDMSR(IA32_TSC_AUX) without vmexit.
- [ ] AC-7: SEV guest: encrypted-VM boots; per-VM SEV key isolated; host cannot read guest memory.
- [ ] AC-8: kvm-unit-tests `svm` test suite passes.
- [ ] AC-9: Live migration: VMCB state checkpoint + restore via KVM_GET_VCPU_EVENTS roundtrips.

## Architecture

`SvmBoot` module-init singleton:

```
struct SvmBoot {
  hsave_pa: PerCpu<u64>,                    // per-CPU host save phys-addr
  msrpm_base: KBox<[u8; MSRPM_BASE_SIZE]>,  // shared zero-default
  iopm_base: KBox<[u8; IOPM_SIZE]>,         // shared zero-default
  npt_enabled: bool,
  avic_enabled: bool,
  asid_max: u32,
  asid_alloc_bitmap: PerCpu<BitmapAlloc>,
}
```

`VcpuSvm` per-vCPU:

```
struct VcpuSvm {
  vcpu: KArc<Vcpu>,
  vmcb: KBox<Vmcb>,                         // current VMCB (vmcb01 or nested.vmcb02)
  vmcb01: KBox<Vmcb>,
  vmcb_pa: u64,                             // physical addr for VMRUN
  asid_generation: u32,
  msrpm: KBox<[u8; MSRPM_SIZE]>,
  current_vmcb: AtomicU64,                  // per-CPU last-loaded (for cache flush)
  host_save: KBox<HostSave>,
  nmi_masked: bool,
  avic_logical_id_table_page: Option<KBox<[u64; 256]>>,
  avic_physical_id_cache: Option<KBox<u64>>,
  ...
}
```

`VcpuSvm::run` flow:
1. `kvm_arch_vcpu_load`: per-CPU VMCB cache flush if needed.
2. Pre-vmenter: load CR0/CR3/CR4 if dirty; load segment regs; restore guest XSAVE; AVIC update if running.
3. `__run` (vmenter.S):
   - Save host GPRs.
   - Load guest GPRs from vcpu.regs.
   - VMSAVE (host_save_pa) — saves additional host state.
   - VMRUN (vmcb_pa) — executes guest until vmexit.
   - VMSAVE (vmcb_pa) — saves guest extras post-vmexit.
   - VMLOAD (host_save_pa) — restores host extras.
   - Save guest GPRs to vcpu.regs.
4. Post-vmexit: read exit_code; check fastpath:
   - If MSR / IPI / VMMCALL fastpath: handle inline.
   - Else: `handle_exit(vcpu, fastpath)`.
5. Return to caller per outcome.

`VcpuSvm::handle_npf` (NPT-fault):
1. Read VMCB.control.exit_info_1 (fault flags) + exit_info_2 (faulting GPA).
2. Build `error_code` from exit_info_1 (P/W/U/X bits).
3. `kvm_mmu_page_fault(vcpu, gpa, error_code, ...)`.
4. If TDP MMU installs SPTE: continue; else: userspace return.

`Avic::init_vcpu(vcpu)` (avic.c):
1. Allocate per-vCPU LAPIC backing page (4KiB; HW writes here on IPI).
2. Per-VM physical-id table entry: write entry[vcpu.apic_id] = (backing_page_pa | valid_bit).
3. Per-VM logical-id table entry: write entry[vcpu.logical_id] = ...
4. VMCB.control.avic_backing_page_pa = backing_page_pa.
5. VMCB.control.avic_logical_id_table_pa = per-VM logical-id-table-pa.
6. VMCB.control.avic_physical_id_table_pa = per-VM physical-id-table-pa.
7. Enable AVIC bit in VMCB.control.

`VcpuSvm::handle_exit` dispatch:
1. exit_code := vmcb.control.exit_code.
2. handler := SVM_EXIT_HANDLERS[exit_code].
3. Call handler; return per outcome.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vmcb_alignment` | INVARIANT | per-VMCB 4KiB-page-aligned; defense against VMRUN #UD on misaligned. |
| `current_vmcb_per_cpu` | INVARIANT | per-CPU current_vmcb tracks last-loaded; defense against stale-VMCB-cache causing wrong-vCPU. |
| `npf_gpa_in_memslot` | INVARIANT | NPT-fault GPA validated against memslot before SPTE install. |
| `vmcb_pa_aligned` | INVARIANT | vmcb_pa physical addr 4KiB aligned. |
| `avic_backing_page_aligned` | INVARIANT | per-vCPU AVIC backing page 4KiB-aligned. |

### Layer 2: TLA+

`virt/kvm/svm_run_loop.tla` (refines vcpu_runloop.tla) models SVM run: state ∈ {NotLoaded, Loaded, Entering, Running, Exiting, Handling}; transitions per VMRUN + vmexit.

`virt/kvm/avic_ipi.tla` models AVIC IPI delivery:
- States per per-vCPU: NotRunning, Running, ReceiveIpi.
- Properties:
  - `safety_running_vcpu_receives_via_doorbell` — IPI to Running vCPU triggers doorbell delivery (no vmexit).
  - `safety_nonrunning_triggers_avic_incomplete_exit` — IPI to NotRunning triggers avic_incomplete_ipi exit on sender.
  - `liveness_ipi_eventually_delivered` — every IPI sent eventually visible in target vAPIC IRR.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuSvm::create` post: vmcb01 allocated; msrpm initialized; avic structures allocated if avic_enabled | `VcpuSvm::create` |
| `VcpuSvm::run` post: vcpu.regs reflect post-vmexit guest state | `VcpuSvm::run` |
| `VcpuSvm::handle_npf` post: SPTE installed iff page available | `VcpuSvm::handle_npf` |
| Per-vCPU vmcb01.asid valid (∈ [1, asid_max]) | `VcpuSvm::create` |
| `Avic::init_vcpu` post: per-VM physical-id-table[apic_id] populated; backing page mapped | `Avic::init_vcpu` |

### Layer 4: Verus/Creusot functional

`VcpuSvm::run loop body → guest instruction execution semantically equivalent to per-instruction emulation`: every guest instruction either (a) vmexits to KVM handler producing same effect as emulator, or (b) executes natively via VMRUN with VMCB-controlled side-effects.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

SVM-specific reinforcement:

- **VMCB cache flush on cross-CPU migrate** — defense against stale per-CPU VMCB cache referencing prior load.
- **Per-vCPU VMCB in IOMMU-protected memory** — defense against device-DMA writing to VMCB.
- **MSRPM initialized restrictive** (all-set = vmexit-on-access) → only allow direct-access for explicitly-listed MSRs.
- **NPT-fault GPA validated** — defense against guest-controlled GPA outside guest-owned memory.
- **AVIC backing page in IOMMU-protected mem** — defense against device-DMA writing AVIC IRR.
- **Per-VM ASID alloc bitmap held under VM lock** — defense against ASID-collision causing TLB-leak across VMs.
- **GIF state-machine via STGI/CLGI** — KVM never sets GIF=1 for guest if KVM has interrupt to inject; defense against missed-IRQ.
- **VMCB.control.intercept_exceptions covers #UD #DB #BP minimum** — defense against guest-injected exception escape.
- **NPT root invalidate on VM-shutdown** — defense against TDP-leak via per-VM ASID reuse.
- **AVIC physical-id-table protection** — only KVM core writes; defense against userspace direct write breaking per-vCPU IPI delivery.
- **Per-VMCB validation pre-VMRUN** (intercept-bitmaps non-null, ASID nonzero, etc.) — defense against malformed VMCB causing #SVME or VMEXIT_INVALID.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — VMCB control area written via host kernel-side struct; KVM_GET/SET_NESTED_STATE bounded by `KVM_STATE_NESTED_SVM_VMCB_SIZE`; no raw user copy into VMCB save area.
- **PAX_KERNEXEC** — `svm_vcpu_run`, `handle_exit`, `nested_svm_vmrun`, `svm_handle_intr` and per-`kvm_x86_ops` SVM slots resolve through RX-only kernel text; ops vtable RO.
- **PAX_RANDKSTACK** — VMRUN entry / VMEXIT handler dispatch inherits RANDKSTACK from KVM_RUN.
- **PAX_REFCOUNT** — per-vCPU VMCB / hsave / nested-state refcount, per-VM ASID counter saturating; concurrent destroy + VMRUN races cannot wrap.
- **PAX_MEMORY_SANITIZE** — per-vCPU VMCB save area, MSR-bitmap, IOIO-bitmap, hsave region zeroed on alloc/free; nested-state buffer zeroed before slab handback so stale L1/L2 register state never leaks.
- **PAX_UDEREF** — KVM_SET_NESTED_STATE copy_from_user STAC/CLAC bracketed; nested VMCB12 GPA validated before load.
- **PAX_RAP / kCFI** — `kvm_x86_ops` SVM slot table RAP-signed; exit-handler dispatch via const `svm_exit_handlers[]` array CFI-guarded.
- **GRKERNSEC_HIDESYM** — VMCB phys-addr, MSR-bitmap kaddr, hsave kaddr, per-vCPU regs[] kaddrs redacted unless CAP_SYSLOG + gr-rbac.
- **GRKERNSEC_DMESG** — VMEXIT_INVALID, malformed-VMCB, AVIC inhibit, GIF-state warnings rate-limited and gated by `dmesg_restrict`.
- **KVM ioctl CAP_SYS_ADMIN strict** — KVM_CREATE_VM (SVM), KVM_SET_NESTED_STATE, KVM_MEMORY_ENCRYPT_OP gated to CAP_SYS_ADMIN + gr-rbac VM-create role.
- **VMCB PAX_USERCOPY** — VMCB control + save area validated pre-VMRUN: intercept-bitmaps non-null, ASID non-zero, EFER consistent, CR0/CR4/CR3 within architectural bounds.
- **MSR-bitmap / IOIO-bitmap host-managed** — guest cannot directly toggle interception; bitmap writes mediated by `svm_set_intercept` under vCPU lock.
- **GIF state-machine** — STGI/CLGI tracked in kernel; KVM never raises GIF=1 with a pending interrupt to inject so the standard `vmcb.control.int_state` cannot drop an IRQ.
- **Nested-virt strict** — L0 enforces VMCB12 consistency (CR0/CR4/EFER, intercept-bitmaps, intercept_exceptions for #UD/#DB/#BP minimum) before allowing L1→L2 transition; L2 cannot bypass L0 by crafting malformed VMCB12.

Per-doc rationale: SVM is one of two hardware virtualization backends; the grsec reinforcement keeps VMCB authoritative kernel-side (no user write into save area), validates VMCB12 pre-nested-VMRUN to defeat L2 escape via malformed VMCB, SANITIZEs MSR/IOIO bitmaps on free, and gates GIF state so an unrooted IRQ never reaches a guest with intercept dropped.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Nested-SVM (covered in `virt/kvm/x86-nested-svm.md` future Tier-3)
- SEV / SEV-ES / SEV-SNP (covered in `virt/kvm/x86-sev.md` future Tier-3)
- AMD PMU (pmu.c covered in `virt/kvm/x86-amd-pmu.md` future Tier-3)
- Hyper-V on SVM (svm_onhyperv.c covered in `virt/kvm/x86-hyperv.md` future Tier-3)
- Intel VMX (covered in `x86-vmx.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- Implementation code
