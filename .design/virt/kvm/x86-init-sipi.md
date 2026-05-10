# Tier-3: arch/x86/kvm/lapic.c (INIT/SIPI subset) — KVM AP boot via INIT + SIPI sequence (per-vCPU mp_state + KVM_MP_STATE_INIT_RECEIVED)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-lapic.md
upstream-paths:
  - arch/x86/kvm/lapic.c (INIT/SIPI; see grep "SIPI\|INIT_RECEIVED\|kvm_apic_set_init")
  - arch/x86/kvm/x86.c (kvm_vcpu_reset on INIT)
  - include/uapi/linux/kvm.h (KVM_MP_STATE_*)
-->

## Summary

x86 SMP brings up Application Processors (APs) via the BSP-issued INIT-then-SIPI (Startup-IPI) sequence: BSP writes ICR with delivery-mode=INIT to AP's APIC ID; APs reset to a known state with mp_state=KVM_MP_STATE_INIT_RECEIVED; BSP then writes ICR with delivery-mode=SIPI + vector=AP's start-page-aligned trampoline; AP starts executing at vector<<12. KVM virtualizes per-vCPU `mp_state` (KVM_MP_STATE_RUNNABLE / _UNINITIALIZED / _INIT_RECEIVED / _SIPI_RECEIVED / _HALTED / _STOPPED). Userspace QEMU drives via KVM_GET/_SET_MP_STATE for live-migration. Critical for: SMP guest boot, hot-add vCPU, suspend/resume.

This Tier-3 covers INIT/SIPI handling in `lapic.c` + `x86.c` (~200-300 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_apic_accept_events(vcpu)` | per-vCPU pending-events processor | `Lapic::accept_events` |
| `kvm_lapic_handle_init(vcpu)` | INIT delivery handler | `Lapic::handle_init` |
| `kvm_lapic_handle_sipi(vcpu, vector)` | SIPI delivery handler | `Lapic::handle_sipi` |
| `kvm_apic_send_init(target_apic, type)` | per-INIT IPI emit | `Lapic::send_init` |
| `vcpu.arch.mp_state` | per-vCPU multiprocessor state | `VcpuArch::mp_state` |
| `kvm_arch_vcpu_ioctl_get_mpstate(vcpu, mp_state)` | KVM_GET_MP_STATE | `Vcpu::ioctl_get_mp_state` |
| `kvm_arch_vcpu_ioctl_set_mpstate(vcpu, mp_state)` | KVM_SET_MP_STATE | `Vcpu::ioctl_set_mp_state` |
| `kvm_vcpu_reset(vcpu, init_event)` | per-vCPU reset on INIT | `Vcpu::reset` |
| `kvm_apic_post_state_restore(vcpu, &s)` | post-state-restore | `Lapic::post_state_restore` |
| `KVM_MP_STATE_RUNNABLE` (0) | running | UAPI |
| `KVM_MP_STATE_UNINITIALIZED` (1) | not yet INIT'ed (BSP-only state) | UAPI |
| `KVM_MP_STATE_INIT_RECEIVED` (2) | INIT received; awaiting SIPI | UAPI |
| `KVM_MP_STATE_HALTED` (3) | HLT executed | UAPI |
| `KVM_MP_STATE_SIPI_RECEIVED` (4) | SIPI received; transitioning to RUNNABLE | UAPI |
| `KVM_MP_STATE_STOPPED` (5) | stopped (s390-specific) | UAPI |
| `kvm_make_request(KVM_REQ_INIT_SIGNAL, vcpu)` | per-INIT request | `Vcpu::make_request` |

## Compatibility contract

REQ-1: x86 INIT-SIPI sequence:
1. BSP writes ICR: vector=0, delivery-mode=INIT, dest=AP-APIC-ID.
2. AP receives INIT-IPI: reset CPU state to known defaults.
3. BSP delays 10ms (real-mode requirement).
4. BSP writes ICR: vector=AP-trampoline-page>>12, delivery-mode=SIPI, dest=AP.
5. AP starts execution at vector<<12 in real-mode.

REQ-2: Per-vCPU `mp_state`:
- INIT signal received: mp_state := INIT_RECEIVED.
- SIPI signal received: mp_state := RUNNABLE; vCPU starts at SIPI vector.

REQ-3: Per-vCPU SMP-init flow:
1. BSP at boot: KVM_RUN; vcpu0 is BSP; all others UNINITIALIZED.
2. BSP issues INIT: per-target vCPU mp_state → INIT_RECEIVED.
3. BSP issues SIPI: per-target vCPU mp_state → SIPI_RECEIVED → RUNNABLE.
4. AP runs from SIPI vector.

REQ-4: `Lapic::handle_init` flow:
1. kvm_vcpu_reset(vcpu, true /* init_event */).
2. mp_state := KVM_MP_STATE_INIT_RECEIVED.
3. kvm_make_request(KVM_REQ_INIT_SIGNAL, vcpu).

REQ-5: Per-vCPU reset on INIT:
- Reset CR0 = 0x60000010 (PE off, CD on, NW on).
- Reset CR3 = 0.
- Reset CR4 = 0.
- Reset segment regs to real-mode values (CS=F000:FFF0, etc.).
- Reset GPRs to 0 (except RDX which contains CPU-id).
- Reset RIP = 0xFFF0.
- Reset EFLAGS = 0x00000002.
- LAPIC reset to default state (preserving APIC ID, version).

REQ-6: `Lapic::handle_sipi` flow:
1. If mp_state != INIT_RECEIVED: drop SIPI (per-spec — ignored).
2. Set CS.base = vector << 12.
3. Set CS.selector = vector << 8.
4. Set RIP = 0.
5. mp_state := KVM_MP_STATE_RUNNABLE.

REQ-7: SIPI delivery target validation:
- Per-spec, SIPI ignored unless target in WAITING-FOR-SIPI state (== INIT_RECEIVED in KVM-speak).
- KVM enforces.

REQ-8: KVM_GET_MP_STATE / _SET_MP_STATE:
- Userspace migration: read + write per-vCPU mp_state.
- For live-migration: serialize per-vCPU INIT/SIPI state.

REQ-9: SMM-state transition coordination:
- SMI ↔ INIT priority per Intel SDM: SMI > INIT.
- KVM defers INIT until SMI handled.

REQ-10: AP wakeup vector (CPUID 0x40000005-style):
- Per-Xen / Hyper-V paravirt: alternative AP-wakeup mechanism.
- Different from INIT-SIPI; covered separately.

REQ-11: Hot-add vCPU:
- Userspace creates vCPU; mp_state = UNINITIALIZED.
- BSP later issues INIT-SIPI; AP joins.

REQ-12: KVM_REQ_INIT_SIGNAL:
- Per-vCPU pending request.
- Processed via kvm_apic_accept_events on vmenter.

## Acceptance Criteria

- [ ] AC-1: 4-vCPU guest boot: BSP issues INIT to vCPU1; mp_state → INIT_RECEIVED.
- [ ] AC-2: SIPI to INIT-pending vCPU: mp_state → RUNNABLE; CS:RIP = vector<<12:0.
- [ ] AC-3: SIPI to RUNNABLE vCPU: ignored (no-op).
- [ ] AC-4: kvm_vcpu_reset on INIT: per-vCPU registers + LAPIC reset to defaults.
- [ ] AC-5: KVM_GET/_SET_MP_STATE: round-trip preserves per-vCPU state.
- [ ] AC-6: SMI-during-INIT: SMI handled first; INIT deferred.
- [ ] AC-7: Hot-add vCPU: mp_state UNINITIALIZED; subsequent INIT-SIPI brings up.
- [ ] AC-8: Live migration: pending INIT_RECEIVED preserved.
- [ ] AC-9: kvm-unit-tests `init_sipi` test passes.
- [ ] AC-10: Multi-AP: 16-vCPU guest brings up all APs via per-AP INIT-SIPI.

## Architecture

`Lapic::accept_events(vcpu)`:
1. apic := vcpu.arch.apic.
2. If !apic.pending_events: return.
3. If pending INIT:
   - Lapic::handle_init(vcpu).
4. If pending SIPI && mp_state == INIT_RECEIVED:
   - Lapic::handle_sipi(vcpu, sipi_vector).
5. Clear pending bits.

`Lapic::handle_init(vcpu)`:
1. kvm_vcpu_reset(vcpu, true).
2. vcpu.arch.mp_state = KVM_MP_STATE_INIT_RECEIVED.

`Lapic::handle_sipi(vcpu, vector)`:
1. If vcpu.arch.mp_state != KVM_MP_STATE_INIT_RECEIVED: return (drop SIPI).
2. cs := { base: vector << 12, selector: vector << 8, limit: 0xFFFF, type: 0x9B, ... }.
3. kvm_arch_set_segment(vcpu, &cs, VCPU_SREG_CS).
4. vcpu.arch.regs[VCPU_REGS_RIP] = 0.
5. vcpu.arch.mp_state = KVM_MP_STATE_RUNNABLE.

`Vcpu::reset(vcpu, init_event)`:
1. CR0 = 0x60000010 (PE=0, CD=1, NW=1, ET=1).
2. CR3 = 0.
3. CR4 = 0.
4. Per-segment reset to BIOS-style real-mode (CS = F000:FFF0).
5. Reset GPRs (RDX = CPUID-family).
6. Reset MSRs to defaults.
7. If init_event: preserve LAPIC base + ID; reset rest.
8. kvm_apic_set_state(vcpu, &reset_state).

`Vcpu::ioctl_get_mp_state(vcpu, mp_state)`:
1. lock vcpu.mutex.
2. mp_state.mp_state = vcpu.arch.mp_state.
3. unlock.

`Vcpu::ioctl_set_mp_state(vcpu, mp_state)`:
1. Validate mp_state.mp_state in valid range.
2. lock vcpu.mutex.
3. vcpu.arch.mp_state = mp_state.mp_state.
4. unlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mp_state_valid` | INVARIANT | per-vCPU mp_state ∈ KVM_MP_STATE_* set. |
| `sipi_only_when_init_received` | INVARIANT | SIPI-handler only acts when mp_state == INIT_RECEIVED. |
| `init_resets_to_known_state` | INVARIANT | per-INIT reset produces per-spec defaults. |
| `cs_base_aligned_for_sipi` | INVARIANT | per-SIPI CS.base = vector << 12 (4KB-aligned). |
| `pending_events_atomic` | INVARIANT | per-vCPU INIT/SIPI events posted under apic.lock. |

### Layer 2: TLA+

`virt/kvm/init_sipi_state.tla`:
- Per-vCPU mp_state transitions.
- Properties:
  - `safety_init_only_when_runnable_or_halted` — INIT signals delivered to RUNNABLE/HALTED vCPUs.
  - `safety_sipi_only_to_init_received` — SIPI-handler ignores non-INIT_RECEIVED state.
  - `liveness_init_sipi_eventually_runnable` — INIT-then-SIPI sequence eventually RUNNABLE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Lapic::handle_init` post: vCPU reset + mp_state == INIT_RECEIVED | `Lapic::handle_init` |
| `Lapic::handle_sipi` post: CS:RIP = vector<<12:0 + mp_state == RUNNABLE | `Lapic::handle_sipi` |
| `Vcpu::reset` post: CR0/CR3/CR4 + segment regs + GPRs match per-spec defaults | `Vcpu::reset` |
| `Vcpu::ioctl_get_mp_state` returns matching mp_state value | `Vcpu::ioctl_get_mp_state` |

### Layer 4: Verus/Creusot functional

`Per-INIT-SIPI sequence: AP transitions UNINIT → INIT_RECEIVED → RUNNABLE; subsequent execution at SIPI vector` semantic equivalence: per-AP boot follows Intel-SDM specification.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-lapic.md` § Hardening.)

INIT/SIPI-specific reinforcement:

- **mp_state validated in valid range** — defense against userspace KVM_SET_MP_STATE causing kernel inconsistency.
- **INIT reset preserves LAPIC ID** — defense against per-AP APIC-ID-loss.
- **SIPI vector validated** — defense against vector causing CS.base OOB.
- **Per-vCPU lock for mp_state** — defense against torn state during concurrent SMM/INIT.
- **SMI > INIT priority** — defense against priority-inversion losing SMI.
- **Per-vCPU reset on INIT atomic** — defense against partial-reset during INIT.
- **Live-migrate per-vCPU mp_state** — defense against post-migrate stuck-AP.
- **Per-spec-compliant: SIPI to non-WAITING vCPU dropped** — defense against deviation.
- **Per-vCPU pending-events under apic.lock** — defense against torn pending state.
- **Hot-add vCPU initial UNINITIALIZED** — defense against pre-INIT execution.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- LAPIC core (covered in `x86-lapic.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- VMX core (covered in `x86-vmx.md` Tier-3)
- SVM core (covered in `x86-svm.md` Tier-3)
- ICR delivery (covered in `x86-lapic.md` Tier-3)
- Hyper-V / Xen AP-wakeup (covered separately)
- Implementation code
