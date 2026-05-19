# Tier-3: arch/x86/kvm/vmx/vmx.c (MTF + guest-debug subset) — VMX Monitor Trap Flag + KVM_GUESTDBG_* + single-step debugging

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-vmx.md
upstream-paths:
  - arch/x86/kvm/vmx/vmx.c (MTF + guest_debug; see grep "MTF\|MONITOR_TRAP_FLAG\|guest_debug")
  - arch/x86/kvm/x86.c (guest_debug ioctl handlers)
  - include/uapi/linux/kvm.h (KVM_GUESTDBG_* + KVM_SET_GUEST_DEBUG)
-->

## Summary

VMX Monitor-Trap Flag (MTF) is the Intel-VMX hardware single-step facility — when CPU-based exec ctrl bit MONITOR_TRAP_FLAG is set, CPU vmexits with reason MONITOR_TRAP_FLAG after each guest instruction; KVM uses this to implement KVM_GUESTDBG_SINGLESTEP for userspace debuggers (gdb stub via QEMU). KVM_SET_GUEST_DEBUG ioctl configures per-vCPU debug-state: KVM_GUESTDBG_ENABLE / _SINGLESTEP / _USE_SW_BP / _USE_HW_BP / _INJECT_DB / _INJECT_BP / _BLOCKIRQ. KVM intercepts #BP (SW breakpoint), #DB (HW breakpoint via DR0-3), and MTF for single-stepping — translates per-event into KVM_EXIT_DEBUG userspace exit. Critical for: gdb-on-VM debugging, kernel-debug remote-vmlinux, virtio-fs probe-debug.

This Tier-3 covers MTF + guest_debug code in `vmx.c` (~200-300 lines) + `x86.c` debug-ioctl handlers.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_arch_vcpu_ioctl_set_guest_debug(vcpu, &dbg)` | KVM_SET_GUEST_DEBUG ioctl | `Vcpu::ioctl_set_guest_debug` |
| `vmx_set_guest_debug(vcpu, dbg)` | per-vendor guest-debug setup | `VcpuVmx::set_guest_debug` |
| `vmx_update_exception_bitmap(vcpu)` | update vmcs.exception-bitmap per debug-state | `VcpuVmx::update_exception_bitmap` |
| `kvm_handle_db_exception(vcpu, ...)` | #DB intercept handler | `Vcpu::handle_db_exception` |
| `kvm_handle_bp_exception(vcpu, ...)` | #BP intercept handler | `Vcpu::handle_bp_exception` |
| `vmx_handle_mtf(vcpu)` | MTF vmexit handler | `VcpuVmx::handle_mtf` |
| `kvm_skip_emulated_instruction(vcpu)` | per-instruction RIP advance + MTF check | `Vcpu::skip_emulated_instruction` |
| `vcpu.guest_debug` | per-vCPU debug-flags field | `Vcpu::guest_debug` |
| `KVM_GUESTDBG_ENABLE` | flag: enable debug | UAPI |
| `KVM_GUESTDBG_SINGLESTEP` | flag: single-step (MTF) | UAPI |
| `KVM_GUESTDBG_USE_SW_BP` | flag: SW breakpoint (#BP intercept) | UAPI |
| `KVM_GUESTDBG_USE_HW_BP` | flag: HW breakpoint (DR0-3 + #DB) | UAPI |
| `KVM_GUESTDBG_INJECT_DB` | flag: inject #DB | UAPI |
| `KVM_GUESTDBG_INJECT_BP` | flag: inject #BP | UAPI |
| `KVM_GUESTDBG_BLOCKIRQ` | flag: block IRQ injection during debug | UAPI |
| `KVM_EXIT_DEBUG` | userspace exit reason | UAPI |
| `kvm_debug_exit_arch` | per-vendor exit-arch info | UAPI |
| `vmcs.GUEST_PENDING_DBG_EXCEPTIONS` | per-vCPU pending #DB info | UAPI |
| `nested_vmx_check_mtf_vmexit(...)` (nested) | nested-VMX MTF | covered in nested.c |

## Compatibility contract

REQ-1: Per-vCPU `guest_debug` flags:
- KVM_GUESTDBG_ENABLE: master-enable.
- KVM_GUESTDBG_SINGLESTEP: per-instruction trap via MTF.
- KVM_GUESTDBG_USE_SW_BP: intercept #BP exceptions to userspace.
- KVM_GUESTDBG_USE_HW_BP: intercept #DB via DR-register configuration.
- KVM_GUESTDBG_INJECT_DB: inject #DB to guest.
- KVM_GUESTDBG_INJECT_BP: inject #BP to guest.
- KVM_GUESTDBG_BLOCKIRQ: suppress IRQ inject during debug.

REQ-2: KVM_SET_GUEST_DEBUG ioctl flow:
1. Validate vcpu.guest_debug = dbg.flags.
2. Configure per-vCPU debug-state:
   - DR0..DR7 from dbg.arch.debugreg.
   - Exception bitmap update.
   - SINGLESTEP: enable MONITOR_TRAP_FLAG in vmcs exec-ctrl.

REQ-3: Per-vCPU exception-bitmap (vmcs.EXCEPTION_BITMAP):
- Bit 1 (#DB): set if KVM_GUESTDBG_USE_HW_BP.
- Bit 3 (#BP): set if KVM_GUESTDBG_USE_SW_BP.
- Other bits per other config.

REQ-4: MTF (Monitor Trap Flag):
- vmcs.cpu_based_exec_control bit MONITOR_TRAP_FLAG.
- When set: vmexit reason MONITOR_TRAP_FLAG after each guest instruction.
- KVM clears MTF after each step + re-sets if SINGLESTEP-mode active.

REQ-5: #DB exception (intercepted in HW_BP-mode):
- Vector 1.
- Triggered by guest code-fetch matching DR0-3 + DR7 enable-bits.
- vmexit reason EXCEPTION_NMI; KVM dispatches to handle_db_exception.
- Returns KVM_EXIT_DEBUG with vcpu.run_state.debug.arch populated.

REQ-6: #BP exception (intercepted in SW_BP-mode):
- Vector 3.
- Triggered by guest INT3 (0xCC) instruction.
- vmexit reason EXCEPTION_NMI; KVM dispatches to handle_bp_exception.
- Returns KVM_EXIT_DEBUG.

REQ-7: Inject #DB / #BP to guest:
- KVM_GUESTDBG_INJECT_DB: queue #DB; per-vmenter VM_ENTRY_INTR_INFO_FIELD = #DB.
- KVM_GUESTDBG_INJECT_BP: queue #BP; per-vmenter inject.

REQ-8: KVM_EXIT_DEBUG return:
- vcpu.run_state.exit_reason = KVM_EXIT_DEBUG.
- vcpu.run_state.debug.arch.exception = (1=#DB, 3=#BP).
- vcpu.run_state.debug.arch.dr6 = (per-spec DR6).
- vcpu.run_state.debug.arch.dr7 = (per-vCPU DR7).
- vcpu.run_state.debug.arch.pc = (RIP).

REQ-9: Single-step semantics:
- Each step: clear MTF; vmenter; immediately vmexit with MTF-reason on next instruction.
- KVM dispatches to userspace via KVM_EXIT_DEBUG.
- Userspace can: continue, step, modify state.

REQ-10: KVM_GUESTDBG_BLOCKIRQ:
- Per-step inhibits IRQ inject so debug-ops not interrupted by IRQs.
- Useful for setting breakpoints in IRQ-handler.

REQ-11: Nested-VMX MTF:
- L1's vmcs12 may set MTF for L2.
- L0's vmcs02 must propagate MTF.
- nested_vmx_check_mtf_vmexit examines + injects via nested.

REQ-12: Per-vmexit MTF-pending state:
- vmx.nested.mtf_pending tracks if L2 vmexit happened mid-MTF processing.
- Required to defer MTF-vmexit-to-L1 until next valid boundary.

## Acceptance Criteria

- [ ] AC-1: KVM_SET_GUEST_DEBUG with KVM_GUESTDBG_ENABLE | _SINGLESTEP: vCPU steps one instruction per vmenter.
- [ ] AC-2: SW breakpoint: guest INT3 → vmexit; KVM_EXIT_DEBUG returned to userspace.
- [ ] AC-3: HW breakpoint via DR0: guest fetches code at DR0-address → #DB → vmexit.
- [ ] AC-4: Inject #DB: KVM_GUESTDBG_INJECT_DB; guest #DB handler invoked.
- [ ] AC-5: BLOCKIRQ: stepping with BLOCKIRQ; IRQs queued not delivered until step complete.
- [ ] AC-6: gdb attach via QEMU: gdb stub commands (continue, step, breakpoint set/clear) work.
- [ ] AC-7: Multi-vCPU debug: 4-vCPU guest; per-vCPU independent debug-state.
- [ ] AC-8: Live migration: in-debug session preserved across migrate.
- [ ] AC-9: kvm-unit-tests `debug` test passes.
- [ ] AC-10: Nested-VMX: L1 debugs L2; L0-MTF + L1-MTF properly cascaded.

## Architecture

`Vcpu::ioctl_set_guest_debug(vcpu, &dbg)`:
1. lock vcpu.mutex.
2. vcpu.guest_debug = dbg.flags.
3. If dbg.flags & KVM_GUESTDBG_USE_HW_BP:
   - For i in 0..4: vcpu.arch.eff_db[i] = dbg.arch.debugreg[i].
   - vcpu.arch.dr7 = dbg.arch.debugreg[7].
4. vmx_set_guest_debug(vcpu, dbg).
5. Update vmcs.EXCEPTION_BITMAP.
6. unlock.

`VcpuVmx::set_guest_debug(vcpu, dbg)`:
1. vmx := to_vmx(vcpu).
2. exception_bitmap := compute new bitmap based on dbg.flags.
3. VMWRITE EXCEPTION_BITMAP = exception_bitmap.
4. If dbg.flags & KVM_GUESTDBG_SINGLESTEP:
   - VMWRITE CPU_BASED_VM_EXEC_CONTROL |= MONITOR_TRAP_FLAG.
5. Else:
   - VMWRITE CPU_BASED_VM_EXEC_CONTROL &= ~MONITOR_TRAP_FLAG.

`VcpuVmx::handle_mtf(vcpu)` (vmexit reason MONITOR_TRAP_FLAG):
1. Clear MTF for next vmenter (one-shot).
2. vcpu.run_state.exit_reason = KVM_EXIT_DEBUG.
3. vcpu.run_state.debug.arch.exception = MTF_VECTOR.
4. vcpu.run_state.debug.arch.pc = vmcs.GUEST_RIP.
5. Return to userspace.

`Vcpu::handle_bp_exception(vcpu)` (vmexit reason EXCEPTION_NMI; vector 3):
1. If !(vcpu.guest_debug & KVM_GUESTDBG_USE_SW_BP):
   - Re-inject #BP to guest.
2. Else:
   - vcpu.run_state.exit_reason = KVM_EXIT_DEBUG.
   - vcpu.run_state.debug.arch.exception = BP_VECTOR.
   - vcpu.run_state.debug.arch.pc = RIP - 1 (pre-INT3).
   - Return to userspace.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dr_idx_bounded` | OOB | per-DR idx < 4; defense against debug-reg array OOB. |
| `mtf_one_shot` | INVARIANT | MTF cleared on vmexit; re-set per-step explicitly. |
| `dbg_flags_validated` | INVARIANT | per-vCPU guest_debug ⊆ KVM_GUESTDBG_VALID_MASK. |
| `exception_bitmap_consistent` | INVARIANT | bitmap reflects per-vCPU debug-state. |
| `inject_db_bp_one_at_a_time` | INVARIANT | inject_db and inject_bp not both pending simultaneously. |

### Layer 2: TLA+

`virt/kvm/mtf_singlestep.tla`:
- Per-vCPU step state ∈ {Idle, Stepping, Stepped}.
- Properties:
  - `safety_one_step_per_vmenter` — Stepping → Stepped after exactly one instruction.
  - `safety_mtf_self_clearing` — MTF cleared automatically per-vmexit.
  - `liveness_step_eventually_completes` — every Stepping eventually Stepped.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::ioctl_set_guest_debug` post: vcpu.guest_debug updated; per-vCPU state matches | `Vcpu::ioctl_set_guest_debug` |
| `VcpuVmx::set_guest_debug` post: vmcs exception-bitmap + MTF reflect dbg.flags | `VcpuVmx::set_guest_debug` |
| `VcpuVmx::handle_mtf` post: MTF cleared; KVM_EXIT_DEBUG returned | `VcpuVmx::handle_mtf` |
| Per-vCPU DR0..7 updated only via guest_debug ioctl when DEBUG enabled | invariants on debug ioctl |

### Layer 4: Verus/Creusot functional

`Per-step: vmenter → CPU executes one instruction → MTF vmexit → userspace KVM_EXIT_DEBUG` semantic equivalence: per-step the resulting RIP advance matches a single instruction's natural semantics.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

MTF / guest-debug-specific reinforcement:

- **MTF one-shot semantics** — defense against MTF-stuck causing infinite vmexit loop.
- **Per-vCPU debug-state mutual-exclusion** — defense against KVM_GUESTDBG_INJECT_DB + INJECT_BP simultaneous.
- **Exception bitmap reflects debug-flags** — defense against intercept-mismatch causing missed debug events.
- **DR0..3 + DR7 validated against guest CR4.DE** — defense against IO-breakpoint config when guest doesn't enable.
- **KVM_GUESTDBG_BLOCKIRQ honored** — defense against IRQ-injection during step disrupting debug semantics.
- **gdb-stub privileged** at userspace — defense against unauthorized debug-attach.
- **Per-vCPU guest_debug atomic** — defense against torn debug-state during ioctl.
- **Nested-MTF cascaded correctly** — defense against L2-MTF leaking to L1 inappropriately.
- **vmcs.GUEST_PENDING_DBG_EXCEPTIONS clear at vmenter** — defense against stale pending #DB.
- **Per-vmexit RIP saved before MTF clear** — defense against losing instruction-completion location.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — KVM_SET_GUEST_DEBUG payload bounded by sizeof(struct kvm_guest_debug).
- **PAX_KERNEXEC** — MTF + debug-step dispatch RO after init.
- **PAX_RANDKSTACK** — randomized kstack per KVM_RUN; #DB save on fresh stack.
- **PAX_REFCOUNT** — guest_debug refcount + pending #DB refcount saturating.
- **PAX_MEMORY_SANITIZE** — guest_debug state + pending-dbg-exceptions zeroed on vCPU destroy.
- **PAX_UDEREF** — KVM_SET_GUEST_DEBUG user pointer validated.
- **PAX_RAP / kCFI** — debug-inject callbacks type-checked.
- **GRKERNSEC_HIDESYM** — RIP / DR0..3 redacted in trace.
- **GRKERNSEC_DMESG** — MTF-storm warnings rate-limited.

MTF-specific:

- **CAP_SYS_ADMIN on KVM_SET_GUEST_DEBUG** — defense against unprivileged debug-attach.
- **MTF one-shot enforced** — defense against infinite vmexit loop.
- **GUEST_PENDING_DBG_EXCEPTIONS cleared at vmenter** — defense against stale #DB carryover.

Rationale: MTF is a single-step primitive that can be abused to time-side-channel host kernel; CAP gate + sanitize on destroy + one-shot enforcement close the cross-VM debug-state leak surface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VMX core (covered in `x86-vmx.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- gdb stub (userspace QEMU concern)
- AMD SVM debug equivalent (covered in `x86-svm.md` Tier-3 + future svm-debug Tier-3)
- Implementation code
