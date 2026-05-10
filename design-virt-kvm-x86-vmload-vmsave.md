---
title: "Tier-3: arch/x86/kvm/svm/svm.c (VMLOAD/VMSAVE subset) — AMD SVM VMLOAD / VMSAVE for additional state save/restore"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

AMD SVM provides VMLOAD / VMSAVE instructions to save/restore additional CPU state not handled by VMRUN — segment register caches (FS/GS/TR/LDTR bases), syscall MSRs (STAR, LSTAR, CSTAR, SFMASK, KernelGsBase), sysenter MSRs (SYSENTER_CS/_ESP/_EIP). Per VMRUN cycle, KVM uses VMSAVE before VMRUN (saves host extra state to per-CPU host_save area) + VMLOAD after VMRUN (restores host extras). Per-vCPU vmcb already has these fields for guest. Critical because VMRUN automatically saves/loads guest's full state but only basic host state (CR/RIP/RSP/RFLAGS/CS/SS/DS/ES); without VMLOAD/VMSAVE, host TR/LDTR/syscall-MSRs would be corrupted.

This Tier-3 covers VMLOAD/VMSAVE in `svm.c` (~150-200 lines) + `vmenter.S` integration.

### Acceptance Criteria

- [ ] AC-1: KVM-AMD module loaded: per-CPU host_save_area allocated.
- [ ] AC-2: Per-VMRUN: VMSAVE before + VMLOAD after; host TR/LDTR/syscall-MSRs preserved.
- [ ] AC-3: Guest VMSAVE/VMLOAD: traps if INTERCEPT_VMSAVE set; emulated by KVM.
- [ ] AC-4: Nested-SVM L1 VMSAVE: L0 emulates → L1's host save-area populated.
- [ ] AC-5: Cross-CPU vCPU migrate: VMLOAD from new-CPU's host_save_area.
- [ ] AC-6: Live migration: vmcb.save state preserved.
- [ ] AC-7: Multi-vCPU: per-vCPU vmcb.save distinct; per-CPU host_save shared.
- [ ] AC-8: SVM v2 LBR: LBR-MSRs round-tripped via VMLOAD/VMSAVE.
- [ ] AC-9: kvm-unit-tests `svm_vmload_vmsave` test passes.
- [ ] AC-10: Performance: per-VMRUN VMLOAD/VMSAVE adds < 100 cycles.

### Architecture

`SvmCpuData::save_area` per-CPU:

```
struct SvmCpuData {
  ...
  save_area: KArc<Page>,                       // 4KiB host_save_area
  save_area_pa: u64,                           // phys addr
}
```

`Vmcb::save` per-vCPU:

```
#[repr(C, packed)]
struct VmcbSaveArea {
  es: VmcbSeg,
  cs: VmcbSeg,
  ss: VmcbSeg,
  ds: VmcbSeg,
  fs: VmcbSeg,
  gs: VmcbSeg,
  gdtr: VmcbSeg,
  ldtr: VmcbSeg,
  idtr: VmcbSeg,
  tr: VmcbSeg,
  cpl: u8,
  efer: u64,
  cr4: u64,
  cr3: u64,
  cr0: u64,
  dr7: u64,
  dr6: u64,
  rflags: u64,
  rip: u64,
  rsp: u64,
  s_cet: u64,
  rax: u64,
  star: u64,
  lstar: u64,
  cstar: u64,
  sfmask: u64,
  kernel_gs_base: u64,
  sysenter_cs: u64,
  sysenter_esp: u64,
  sysenter_eip: u64,
  cr2: u64,
  g_pat: u64,
  dbgctl: u64,
  br_from: u64,
  br_to: u64,
  last_excp_from: u64,
  last_excp_to: u64,
  ...
}
```

`Svm::pre_vmrun` (in vmenter.S):
```asm
vmsave [rcx]    ; rcx = host_save_pa
vmrun [rax]     ; rax = vmcb_pa
; ...vmexit returns here...
vmsave [rax]    ; save guest extras
vmload [rcx]    ; restore host extras
```

`Svm::setup_save_area(svm, init_event)` (per-vCPU vmcb init):
1. save := &svm.vmcb01.save.
2. save.cs := real-mode-CS.
3. save.efer := EFER_SVME.
4. save.cr0 := CR0_PE | CR0_NE.
5. save.cr4 := 0.
6. save.idtr.limit := 0xFFFF.
7. save.gdtr.limit := 0xFFFF.
8. save.dr7 := 0x400.
9. save.tr := { selector: 0, base: 0, limit: 0xFFFF, type: TR }.
10. save.ldtr := { selector: 0, base: 0, limit: 0xFFFF, type: LDTR }.

`Svm::handle_vmsave(vcpu)` (intercept-based emulation):
1. vmcb_gpa := vcpu.regs[VCPU_REGS_RAX].
2. Validate vmcb_gpa within memslot.
3. Read guest current state.
4. Write to vmcb_gpa per VMSAVE semantics.
5. skip_emulated_instruction.

`Svm::handle_vmload(vcpu)` (intercept-based):
1. vmcb_gpa := vcpu.regs[VCPU_REGS_RAX].
2. Validate.
3. Read save-area from vmcb_gpa.
4. Write per-VMLOAD fields to current state.
5. skip_emulated_instruction.

### Out of Scope

- SVM core (covered in `x86-svm.md` Tier-3)
- SVM vmenter (covered in `x86-svm-vmenter.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- Nested-SVM (covered in `x86-nested-svm.md` Tier-3)
- AMD APM specification details (vendor doc)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `__svm_vcpu_run(vcpu)` | per-vCPU run loop with VMLOAD/VMSAVE | covered in `x86-svm-vmenter.md` |
| `svm_cpu_data.save_area` | per-CPU host_save_area | `SvmCpuData::save_area` |
| `svm_cpu_data.save_area_pa` | physical addr | `SvmCpuData::save_area_pa` |
| `vmcb.save` (struct vmcb_save_area) | per-vCPU full save area | `Vmcb::save` |
| `init_vmcb(svm, init_event)` | per-VM initial vmcb populate | covered in `x86-svm.md` |
| `svm_setup_save_area(svm, ...)` | per-vCPU vmcb.save init | `Svm::setup_save_area` |
| `svm_save_host_state(...)` | (where applicable) per-vCPU host save | shared with vmcb01 |
| `svm_setup_msr_pass_through(svm)` | per-vCPU MSR-bitmap setup | covered in `x86-msr.md` |
| `INTERCEPT_VMSAVE` / `_VMLOAD` | vmcb intercept bits | UAPI |
| `vmcb.control.intercept` per-spec bits | per-VM intercept | UAPI |

### compatibility contract

REQ-1: `vmcb.save` (vmcb_save_area struct):
- `es / cs / ss / ds / fs / gs / gdtr / idtr / ldtr / tr` (per-segment cached descriptor).
- `cpl` (current privilege level).
- `efer / cr4 / cr3 / cr0 / cr2`.
- `rip / rsp / rax / rflags / star / lstar / cstar / sfmask / kernel_gs_base`.
- `sysenter_cs / sysenter_esp / sysenter_eip / dr6 / dr7`.
- `g_pat`.
- `dbgctl / br_from / br_to / last_excp_from / last_excp_to`.

REQ-2: VMSAVE instruction:
- Operand: physical addr of save-area (4KB aligned).
- Saves per-CPU current state to save-area.
- Specifically: FS / GS / TR / LDTR bases + syscall MSRs + sysenter MSRs.

REQ-3: VMLOAD instruction:
- Operand: physical addr.
- Loads from save-area into CPU.
- Same fields as VMSAVE.

REQ-4: Per-VMRUN flow with VMLOAD/VMSAVE:
1. Pre-VMRUN: VMSAVE host_save_area (saves host extras).
2. VMRUN vmcb_pa (CPU: saves basic host + loads guest from vmcb).
3. Guest executes; vmexits.
4. Post-VMRUN: VMSAVE vmcb01_pa (saves guest extras into vmcb).
5. VMLOAD host_save_area (restores host extras).

REQ-5: Per-CPU host_save_area:
- Allocated at module init per-CPU.
- Persisted across VMRUN cycles.
- Avoids per-VMRUN alloc/free.

REQ-6: VMSAVE/VMLOAD interception:
- vmcb.intercept INTERCEPT_VMSAVE / _VMLOAD bits.
- When set: guest VMSAVE/VMLOAD vmexits.
- Default: NOT intercepted (host's VMLOAD/VMSAVE around VMRUN doesn't trap; guest's would if set).
- Used: nested-SVM (L1 may execute VMSAVE/VMLOAD).

REQ-7: VMRUN auto-save (basic):
- VMRUN saves: CR0/CR3/CR4/EFER, RFLAGS, RIP, RSP, RAX, CS, SS, DS, ES, GDTR, IDTR.
- Loads same from vmcb.save area.
- Does NOT auto-save: TR/LDTR/FS/GS/syscall-MSRs/sysenter-MSRs.

REQ-8: Per-CPU MSRs cleanup post-VMRUN:
- Some host MSRs may need explicit re-load (KERNEL_GS_BASE, etc.).
- VMLOAD covers most; per-arch additional may be needed.

REQ-9: Nested-SVM VMSAVE/VMLOAD:
- L1 may execute VMSAVE/VMLOAD on L2's vmcb.
- L0 may intercept; emulate by save-area-copy.

REQ-10: AMD SVM v2 LBR virtualization:
- VMSAVE/VMLOAD include LBR-related MSRs (br_from / br_to / last_excp_*) when LBR-active.

REQ-11: Per-vCPU vmcb01 vs vmcb02 (nested):
- vmcb01: L1's vmcb (host runs L1).
- vmcb02: L2's vmcb (L1 runs L2 via nested-SVM).
- Per-VMSAVE selects appropriate vmcb_pa.

REQ-12: Live migration:
- Per-vCPU vmcb.save state migrated.
- Per-CPU host_save_area not migrated (per-host transient).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `host_save_area_per_cpu_allocated` | INVARIANT | per-CPU save_area allocated at SvmCpuData init. |
| `save_area_aligned_4k` | INVARIANT | host_save_pa + vmcb_pa 4KiB-aligned per-spec. |
| `vmsave_vmload_paired` | INVARIANT | every VMSAVE host_save paired with VMLOAD host_save (per-VMRUN cycle). |
| `nested_vmsave_intercept_consistent` | INVARIANT | INTERCEPT_VMSAVE bit set ↔ guest VMSAVE traps. |
| `vmcb_save_field_layout_match` | INVARIANT | vmcb.save struct layout matches AMD APM spec. |

### Layer 2: TLA+

`virt/kvm/svm_vmload_vmsave.tla`:
- Per-VMRUN cycle state.
- Properties:
  - `safety_host_state_preserved` — per-VMRUN cycle: post-VMRUN host TR/LDTR/syscall-MSRs match pre-VMRUN.
  - `safety_guest_state_preserved` — per-VMRUN cycle: guest extras saved + restored from vmcb.save.
  - `liveness_vmload_eventually_completes` — VMLOAD completes within bounded cycles.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Svm::pre_vmrun` post: host extras saved to host_save_area | implicit in vmenter.S |
| `Svm::post_vmexit` post: guest extras saved to vmcb01.save; host extras restored from host_save | implicit |
| `Svm::handle_vmsave` post: guest VMSAVE emulated; per-spec save-area populated | `Svm::handle_vmsave` |
| `Svm::handle_vmload` post: guest VMLOAD emulated; per-spec state loaded | `Svm::handle_vmload` |

### Layer 4: Verus/Creusot functional

`Per-VMRUN cycle: host TR/LDTR/syscall-MSRs preserved across guest execution; guest extras correctly saved + restored` semantic equivalence: per-cycle full extra-state round-trip.

### hardening

(Inherits row-1 features from `virt/kvm/x86-svm.md` § Hardening.)

VMLOAD/VMSAVE-specific reinforcement:

- **Per-CPU host_save_area pinned** — defense against host-paging out save_area mid-VMRUN.
- **VMSAVE/VMLOAD addr 4KiB-aligned** — defense against #UD on misaligned.
- **VMSAVE/VMLOAD around VMRUN** — defense against host extras corruption.
- **Per-vmcb.save layout per AMD APM** — defense against field-layout-mismatch causing incorrect state.
- **INTERCEPT_VMSAVE for nested-SVM** — defense against L1 directly modifying L2 state.
- **vmcb_pa validated against guest memslot** for guest VMSAVE/VMLOAD — defense against host kernel-mem write.
- **Per-VMRUN host_save persistent across cycles** — defense against per-cycle alloc overhead.
- **Save-area access under preempt-disabled** — defense against host-IRQ between save+VMRUN corrupting state.
- **Per-vCPU vmcb01 vs vmcb02 distinct save** — defense against L1/L2 state cross-contamination.
- **SVM v2 LBR fields included** — defense against LBR-state-loss when active.

