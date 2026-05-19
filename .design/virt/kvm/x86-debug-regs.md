# Tier-3: arch/x86/kvm/x86.c (DR subset) — KVM debug register virtualization (DR0-3 + DR6 + DR7 + per-vCPU dr-bitmap)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/x86.c (DR sections; see grep "kvm_set_dr|kvm_get_dr|DR0|DR7")
  - arch/x86/kvm/vmx/vmx.c (vmx_set_dr|GUEST_DR7)
  - arch/x86/kvm/svm/svm.c (svm_set_dr)
-->

## Summary

x86 debug registers DR0-3 (breakpoint addresses), DR6 (debug status), DR7 (debug control) provide hardware breakpoint capability per CPU. KVM virtualizes these per-vCPU: when guest enables DR0-3 + DR7 (CR0.DE = 1; bit-Lx in DR7 set), per-instruction-fetch the CPU compares against DR0-3 and signals #DB on match. KVM intercepts DR access via vmcs.exec_control bit MOV_DR_EXITING + vmcb.intercept bit MOV_DR. Per-vCPU `vcpu.arch.eff_db[4]` + `vcpu.arch.dr7` track effective values; KVM_SET_GUEST_DEBUG controls per-vCPU policy (covered in mtf.md). Per-#DB exception decode: DR6 indicates which breakpoint matched (B0-B3, BD, BS, BT bits).

This Tier-3 covers DR-virtualization in `arch/x86/kvm/x86.c` (~150-200 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_set_dr(vcpu, dr, val)` | per-vCPU DR write | `Vcpu::set_dr` |
| `kvm_get_dr(vcpu, dr)` | per-vCPU DR read | `Vcpu::get_dr` |
| `kvm_handle_invalid_op(vcpu)` (#UD) | per-DR4/5 access (deprecated) | `Vcpu::handle_invalid_op` |
| `vcpu.arch.db[4]` | per-vCPU DR0-3 shadow | `VcpuArch::db` |
| `vcpu.arch.dr6` | per-vCPU DR6 shadow | `VcpuArch::dr6` |
| `vcpu.arch.dr7` | per-vCPU DR7 shadow | `VcpuArch::dr7` |
| `vcpu.arch.eff_db[4]` | per-vCPU effective DR0-3 (HW values; may differ from db[] if KVM_GUESTDBG_USE_HW_BP) | `VcpuArch::eff_db` |
| `vcpu.arch.guest_debug_dr7` | per-vCPU host-debug-managed DR7 | `VcpuArch::guest_debug_dr7` |
| `vmx_sync_dirty_debug_regs(vcpu)` (vmx.c) | per-vmexit dirty DR sync | `VcpuVmx::sync_dirty_debug_regs` |
| `vmx_set_dr7(vcpu, val)` | VMX vmcs.GUEST_DR7 write | `VcpuVmx::set_dr7` |
| `vmx_get_dr7(vcpu)` | VMX vmcs.GUEST_DR7 read | `VcpuVmx::get_dr7` |
| `update_exception_bitmap(vcpu)` | per-vCPU exception-bitmap refresh (#DB intercept) | `VcpuVmx::update_exception_bitmap` |
| `vmcs.GUEST_DR7` | per-vCPU HW DR7 | UAPI |
| `vmcb.save.dr6` / `_dr7` (SVM) | per-vCPU SVM DR fields | UAPI |

## Compatibility contract

REQ-1: Per-vCPU DR shadow:
- `vcpu.arch.db[4]`: guest-visible DR0-3.
- `vcpu.arch.dr6`: guest-visible DR6.
- `vcpu.arch.dr7`: guest-visible DR7.

REQ-2: Per-vCPU `eff_db[4]`:
- HW-loaded DR0-3 values.
- Same as db[] when KVM_GUESTDBG_USE_HW_BP off.
- Distinct when KVM_GUESTDBG_USE_HW_BP on (host-debug uses HW; guest sees its own).

REQ-3: Per-vCPU `guest_debug_dr7`:
- Per-vCPU DR7 used by KVM for HW breakpoint set via KVM_SET_GUEST_DEBUG.
- ORed with vcpu.arch.dr7 to compute vmcs.GUEST_DR7.

REQ-4: DR0-3 (breakpoint addresses):
- 64-bit each.
- Linear address triggering #DB on match.
- Match conditions controlled by DR7.

REQ-5: DR6 (debug status):
- Bit[0] B0 — DR0 breakpoint matched.
- Bit[1] B1.
- Bit[2] B2.
- Bit[3] B3.
- Bit[13] BD (debug-register-access).
- Bit[14] BS (single-step).
- Bit[15] BT (task-switch).
- Bits[16..31] reserved (must be 0).

REQ-6: DR7 (debug control):
- Bit[0..7] L0/G0/L1/G1/L2/G2/L3/G3 (per-DRn local + global enable).
- Bit[8] LE (Local Exact); Bit[9] GE (Global Exact).
- Bit[10] always 1.
- Bit[13] GD (General Detect; trap on DR access).
- Bits[16..31]: per-DRn LEN (length: 1/2/4/8 byte) + R/W (00=instr, 01=write, 10=I/O, 11=read-write).

REQ-7: Per-DR-write `kvm_set_dr(vcpu, dr, val)` flow:
1. If DR4 or DR5 + CR4.DE = 1: kvm_inject_gp(vcpu, 0); return.
2. If DR4 / DR5: alias to DR6 / DR7.
3. Switch dr:
   - 0..3: vcpu.arch.db[dr] = val.
   - 6: validate reserved-bits zero; vcpu.arch.dr6 = val.
   - 7: validate reserved-bits zero; vcpu.arch.dr7 = val.
4. If dr == 7: kvm_update_dr7(vcpu) → recomputes effective DR7.

REQ-8: Per-DR-read `kvm_get_dr(vcpu, dr)`:
1. Switch dr:
   - 0..3: return vcpu.arch.db[dr].
   - 4 / 6: return vcpu.arch.dr6.
   - 5 / 7: return vcpu.arch.dr7.

REQ-9: VMX MOV_DR_EXITING:
- vmcs.exec_control bit 23.
- When set: every guest MOV-DR access vmexits.
- VMEXIT_REASON_DR_ACCESS handler decodes register + emulates via kvm_set_dr/_get_dr.

REQ-10: VMX vmcs.GUEST_DR7:
- Per-vmenter HW DR7.
- Updated on each kvm_set_dr(7) or guest_debug change.

REQ-11: Per-vCPU effective DR7 computation:
- If KVM_GUESTDBG_USE_HW_BP: GUEST_DR7 = vcpu.arch.guest_debug_dr7.
- Else: GUEST_DR7 = vcpu.arch.dr7.

REQ-12: Per-vmexit dirty-DR sync:
- If guest modified DR0-3 or DR6 during execution: vmcs/vmcb may need re-read.
- vmx_sync_dirty_debug_regs handles per-vmexit.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest: kernel can MOV CR4 to set DE=1; subsequent MOV DR0..3 valid.
- [ ] AC-2: Per-DRx write: vcpu.arch.db[x] updated; subsequent guest MOV DR returns same.
- [ ] AC-3: DR4/DR5 with DE=1: #GP injected.
- [ ] AC-4: DR6 reserved bits write: rejected via #GP or normalized.
- [ ] AC-5: KVM_SET_GUEST_DEBUG with KVM_GUESTDBG_USE_HW_BP: HW DR0-3 set; guest sees its own.
- [ ] AC-6: HW breakpoint trigger: #DB raised; DR6.B0 set.
- [ ] AC-7: Single-step (DR7.GD): per-instruction #DB; DR6.BS set.
- [ ] AC-8: Live migration: DR state preserved.
- [ ] AC-9: Per-vCPU SMP: per-vCPU DR distinct.
- [ ] AC-10: kvm-unit-tests `dr_access` test passes.

## Architecture

`Vcpu::set_dr(vcpu, dr, val)`:
1. If (dr == 4 OR dr == 5) && (kvm_read_cr4_bits(vcpu, X86_CR4_DE)): 
   - kvm_inject_gp(vcpu, 0).
   - return -1.
2. If dr == 4: dr = 6.
3. Else if dr == 5: dr = 7.
4. Switch dr:
   - 0: vcpu.arch.db[0] = val; mark_dr_dirty.
   - 1: vcpu.arch.db[1] = val.
   - 2: vcpu.arch.db[2] = val.
   - 3: vcpu.arch.db[3] = val.
   - 6:
     - reserved := 0xFFFFFFFF_FFFE_0FF0.
     - If val & reserved && !host_initiated:
       - kvm_inject_gp(vcpu, 0).
       - return -1.
     - val := (val & ~DR_TRAP_BITS) | (vcpu.arch.dr6 & DR_TRAP_BITS).  // preserve B0-B3
     - vcpu.arch.dr6 = val.
   - 7:
     - reserved := 0xFFFFFFFF_0000_0400.
     - If val & reserved && !host_initiated:
       - kvm_inject_gp(vcpu, 0).
       - return -1.
     - vcpu.arch.dr7 = val.
     - kvm_update_dr7(vcpu).
5. Return 0.

`Vcpu::get_dr(vcpu, dr)`:
1. If dr == 4: dr = 6.
2. Else if dr == 5: dr = 7.
3. Switch dr:
   - 0: return vcpu.arch.db[0].
   - 1: return vcpu.arch.db[1].
   - 2: return vcpu.arch.db[2].
   - 3: return vcpu.arch.db[3].
   - 6: return vcpu.arch.dr6.
   - 7: return vcpu.arch.dr7.

`Vcpu::update_dr7(vcpu)`:
1. eff_dr7 := vcpu.arch.dr7.
2. If KVM_GUESTDBG_USE_HW_BP:
   - eff_dr7 |= vcpu.arch.guest_debug_dr7.
3. Vendor: vmx_set_dr7(vcpu, eff_dr7) OR svm_set_dr7(vcpu, eff_dr7).

`VcpuVmx::sync_dirty_debug_regs(vcpu)`:
1. If !(vcpu.guest_debug & KVM_GUESTDBG_USE_HW_BP):
   - // guest may have modified DR0-3 / DR6.
   - get_dr_dirty: read vmcs/vmcb to refresh shadow.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dr_idx_bounded` | OOB | per-DR idx ∈ [0, 7]; defense against array OOB. |
| `dr6_reserved_bits_zero` | INVARIANT | per-vCPU dr6 reserved bits zero. |
| `dr7_reserved_bits_zero` | INVARIANT | per-vCPU dr7 reserved bits zero. |
| `de_required_for_dr4_dr5` | INVARIANT | DR4/DR5 access valid iff CR4.DE = 1; else aliased to DR6/DR7. |
| `dr7_bit_10_always_set` | INVARIANT | per-spec DR7 bit-10 must be 1. |

### Layer 2: TLA+

`virt/kvm/dr_access.tla`:
- Per-vCPU DR state.
- Properties:
  - `safety_dr4_dr5_aliased_when_de_clear` — DR4/DR5 access without CR4.DE aliases to DR6/DR7.
  - `safety_dr6_b0_set_on_break_match` — per-#DB the matching B0-B3 bit set.
  - `safety_per_vCPU_dr_isolated` — per-vCPU DR distinct.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::set_dr` post: per-DR shadow updated; vmcs reflected for DR7 | `Vcpu::set_dr` |
| `Vcpu::get_dr` post: returned value matches shadow | `Vcpu::get_dr` |
| `Vcpu::update_dr7` post: vmcs.GUEST_DR7 OR vmcb.dr7 reflects effective DR7 | `Vcpu::update_dr7` |
| Per-vCPU eff_db distinct from db when KVM_GUESTDBG_USE_HW_BP active | invariants on guest_debug |

### Layer 4: Verus/Creusot functional

`Per-vCPU DR-write: subsequent guest MOV-DR-read returns written value; HW breakpoint matches per-DR0-3 + DR7-config` semantic equivalence: per-vCPU DR semantics match Intel SDM.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

DR-virtualization-specific reinforcement:

- **Per-DR reserved bits enforced** — defense against guest writing reserved bits causing CPU exception.
- **DR4/DR5 alias gating on CR4.DE** — defense against incorrect alias when DE clear.
- **Per-vCPU effective DR7 atomic** — defense against torn DR7 read during cross-CPU access.
- **Per-vmexit dirty DR sync** — defense against stale shadow after guest direct write.
- **HW BP (KVM_GUESTDBG_USE_HW_BP) isolated from guest DR** — defense against guest tampering with host-managed BP.
- **dr6.B0-B3 bits preserved on write** — defense against guest losing exception-history.
- **MOV_DR_EXITING tied to KVM_GUESTDBG_* flags** — defense against intercept-bypass.
- **Per-vCPU DR migrated** — defense against post-migrate stale DR.
- **#DB exception priority correct** — defense against MTF/INT3/etc. priority confusion.
- **Per-DR access privileged in nested-VMX** — defense against L2 reading L1's DR.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — KVM_GET/SET_DEBUGREGS copy bounded by sizeof(struct kvm_debugregs).
- **PAX_KERNEXEC** — DR read/write dispatch RO after init.
- **PAX_RANDKSTACK** — randomized kstack per KVM_RUN; #DB save/restore on fresh stack.
- **PAX_REFCOUNT** — host HW-BP slot reservation refcount saturating.
- **PAX_MEMORY_SANITIZE** — vCPU effective DR0–DR7 zeroed on vCPU destroy; no DR leak across VMs.
- **PAX_UDEREF** — KVM_GET/SET_DEBUGREGS user pointer validated.
- **PAX_RAP / kCFI** — set_dr / get_dr indirect-call type-checked.
- **GRKERNSEC_HIDESYM** — DR0–DR3 (guest BP addresses) redacted in trace.
- **GRKERNSEC_DMESG** — DR-intercept floods rate-limited.

DR-specific:

- **CAP_SYS_ADMIN strict on KVM_SET_GUEST_DEBUG / KVM_SET_DEBUGREGS** — defense against unprivileged BP injection.
- **HW-BP slots namespaced per-VM, accounted refcounted** — defense against per-VM slot exhaustion.
- **DR-content sanitize on live-migrate destroy** — no breakpoint-address leak across migrations.

Rationale: guest DR holds breakpoint VAs that can fingerprint guest binaries; sanitize on destroy + UDEREF on ioctl payload prevents cross-VM DR-leak fingerprinting.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- VMX core (covered in `x86-vmx.md` Tier-3)
- SVM core (covered in `x86-svm.md` Tier-3)
- KVM_SET_GUEST_DEBUG (covered in `x86-mtf.md` Tier-3)
- Implementation code
