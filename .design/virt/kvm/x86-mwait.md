# Tier-3: arch/x86/kvm/vmx/vmx.c (MWAIT/PAUSE subset) — KVM MWAIT/MONITOR/PAUSE intercept handling + idle-detection

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-vmx.md
upstream-paths:
  - arch/x86/kvm/vmx/vmx.c (mwait/pause; see grep "handle_mwait|handle_pause|handle_monitor|MWAIT_EXITING|PAUSE_EXITING")
  - arch/x86/kvm/x86.c (kvm_vcpu_halt + idle integration)
  - include/uapi/linux/kvm.h (KVM_X86_DISABLE_EXITS)
-->

## Summary

KVM virtualizes guest-side MWAIT / MONITOR / PAUSE instructions for guest idle-loops + spinlock-yield. By default KVM intercepts MWAIT (vmcs.exec_control bit MWAIT_EXITING) — guest MWAIT vmexits + becomes effectively HLT (vCPU halts; KVM schedules out). MONITOR similarly intercepted. PAUSE is rate-monitored via vmcs.PAUSE_EXITING + window/threshold to detect spinning + dispatch yield-to-other-vCPU. Per-VM KVM_X86_DISABLE_EXITS option disables these intercepts (guest runs MWAIT/PAUSE natively; risky if untrusted guest can hog CPU). Critical for: Linux idle (MWAIT-based mwait_idle), pthread_spin_lock, qspinlock guest fast-path.

This Tier-3 covers MWAIT/MONITOR/PAUSE handling in `vmx.c` (~150-200 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `handle_mwait(vcpu)` | per-MWAIT vmexit | `VcpuVmx::handle_mwait` |
| `handle_monitor(vcpu)` | per-MONITOR vmexit | `VcpuVmx::handle_monitor` |
| `handle_pause(vcpu)` | per-PAUSE vmexit | `VcpuVmx::handle_pause` |
| `kvm_vcpu_halt(vcpu)` | per-vCPU halt + schedule-out | `Vcpu::halt` |
| `kvm_vcpu_block(vcpu)` | per-vCPU sleep | `Vcpu::block` |
| `kvm_vcpu_yield_to(target)` | per-vCPU yield-to-other | `Vcpu::yield_to` |
| `kvm_vcpu_yield_loop_yield(vcpu)` | per-vCPU yield-loop | `Vcpu::yield_loop_yield` |
| `vmx_set_intercept_for_msr(...)` MWAIT-related | per-MWAIT-MSR intercept | covered in `x86-msr.md` |
| `KVM_X86_DISABLE_EXITS` | per-VM intercept-disable mask | UAPI |
| `MWAIT_EXITING` (CPU_BASED_EXEC_CTRL bit) | vmcs MWAIT vmexit-enable | UAPI |
| `PAUSE_EXITING` | vmcs PAUSE vmexit-enable | UAPI |
| `MONITOR_TRAP_FLAG_*` | not MWAIT-related; covered in mtf.md | shared |
| `kvm_emulate_pause(vcpu)` | PAUSE emulation | `Vcpu::emulate_pause` |

## Compatibility contract

REQ-1: MWAIT instruction:
- Guest-side: load address into RAX (target memory), execute MONITOR; load wake-state into ECX/EAX; execute MWAIT.
- HW behavior: CPU enters wait-state; wakes on memory-write to monitored addr OR external event.

REQ-2: PAUSE instruction:
- Hint to CPU: "spinning"; reduces power + improves SMT thread-yield.
- Used in spinlock fast-path; modern Linux qspinlock spin-loop.

REQ-3: KVM intercept policy:
- Default: MWAIT_EXITING + MONITOR_EXITING set; PAUSE_EXITING gated by per-VM config.
- Per-VM KVM_X86_DISABLE_EXITS:
  - KVM_X86_DISABLE_EXITS_HLT: don't intercept HLT.
  - KVM_X86_DISABLE_EXITS_MWAIT: don't intercept MWAIT.
  - KVM_X86_DISABLE_EXITS_PAUSE: don't intercept PAUSE.

REQ-4: handle_mwait flow:
1. If !KVM_X86_DISABLE_EXITS_MWAIT: 
   - kvm_emulate_halt(vcpu) — treat as HLT.
2. Else:
   - skip_emulated_instruction.
   - Fall through to next vmenter; CPU executes MWAIT natively.

REQ-5: handle_monitor flow:
1. If !KVM_X86_DISABLE_EXITS_MWAIT (controls MONITOR jointly):
   - skip_emulated_instruction.
   - Treat as no-op (KVM doesn't track per-vCPU MONITOR state).

REQ-6: handle_pause flow (PLE: PAUSE-Loop-Exiting):
1. PLE-window VMCS field configures: max consecutive PAUSEs allowed without forward-progress.
2. PLE-gap VMCS field: time-quantum.
3. Per-PLE-vmexit:
   - vcpu.arch.pause_in_guest counter advances.
   - kvm_vcpu_yield_to identify spinning-vCPU; yield host CPU.
4. skip_emulated_instruction; resume vCPU.

REQ-7: PLE goal:
- Detect guest-spinlock-spin scenario (vCPU N spins waiting for vCPU M which is preempted).
- Yield N's host-CPU to M so M can release lock; reduces wasteful spin-cycles.

REQ-8: Per-vCPU yield-loop tracking:
- vcpu.arch.in_yield_loop track if currently yielding.
- Avoid yield-loop-feedback (vCPU keeps yielding to itself).

REQ-9: kvm_vcpu_halt:
- Guest HLT or MWAIT-treated-as-HLT.
- Disables IRQ.
- Sets vcpu.run_state = HALTED.
- schedule().
- On wake (IRQ pending OR KVM_REQ_UNHALT): vCPU resumes.

REQ-10: KVM_X86_DISABLE_EXITS gating:
- Userspace QEMU explicitly opts-in.
- Risk: untrusted guest can hold host CPU indefinitely via unbounded-MWAIT.
- Production: require per-VM CPU-pinning + cgroup-cpuset limits before enable.

REQ-11: AMD SVM analog:
- vmcb.intercept bit INTERCEPT_MWAIT / _MONITOR / _PAUSE.
- Same intercept policy.

REQ-12: Per-VM PV-features integration:
- KVM_FEATURE_PV_UNHALT: guest qspinlock uses kvm-pv-yield instead of MWAIT/PAUSE.
- Eliminates MWAIT vmexit on lock-acquire-spin.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest: cpuidle uses mwait_idle; per-MWAIT triggers vmexit → kvm_vcpu_halt.
- [ ] AC-2: Disable MWAIT exits: KVM_X86_DISABLE_EXITS_MWAIT; subsequent MWAIT runs natively (no vmexit).
- [ ] AC-3: PLE: 16-vCPU guest with heavy spinlock contention; PAUSE_EXITING dispatches yield_to.
- [ ] AC-4: Per-vCPU yield: vCPU N yields to vCPU M when M holds lock + N spins.
- [ ] AC-5: KVM_FEATURE_PV_UNHALT: guest enables; subsequent qspinlock-contention uses kvm-pv-yield not MWAIT.
- [ ] AC-6: Spec-violation defense: enabling DISABLE_EXITS_MWAIT on shared-CPU VM logs warning.
- [ ] AC-7: PLE-window tunable via module param: ple_window (default 4096); ple_gap (default 128).
- [ ] AC-8: HLT in-guest: KVM_X86_DISABLE_EXITS_HLT; vCPU HLT runs natively; reduced vmexit count.
- [ ] AC-9: Per-vCPU pause stat: /sys/kernel/debug/kvm/<id>/<vcpu>/pause_count visible.
- [ ] AC-10: kvm-unit-tests `pause` test passes.

## Architecture

`VcpuVmx::handle_mwait(vcpu)`:
1. If kvm.arch.mwait_in_guest: 
   - skip_emulated_instruction.
   - return 1.
2. Else:
   - return kvm_emulate_halt(vcpu).

`VcpuVmx::handle_monitor(vcpu)`:
1. If kvm.arch.mwait_in_guest:
   - skip_emulated_instruction.
   - return 1.
2. Else:
   - skip_emulated_instruction; treat as no-op.
   - return 1.

`VcpuVmx::handle_pause(vcpu)`:
1. If kvm.arch.pause_in_guest:
   - skip_emulated_instruction.
   - return 1.
2. Else:
   - kvm_emulate_pause(vcpu).
   - kvm_vcpu_on_spin(vcpu, vcpu.arch.cpl == 0).
   - skip_emulated_instruction.
   - return 1.

`Vcpu::halt(vcpu)`:
1. trace_kvm_vcpu_halt(vcpu).
2. vcpu.run_state.exit_reason = KVM_EXIT_HLT (if userspace HLT enabled).
3. Else: vcpu.arch.mp_state = KVM_MP_STATE_HALTED.
4. schedule_out (vcpu thread sleeps).
5. On wake: kvm_vcpu_running checks pending events; resumes.

`Vcpu::yield_to(target)`:
1. If !target OR target == vcpu: skip.
2. yield_to(target.task, false).
3. Returns 1 if scheduler yielded.

`Vcpu::yield_loop_yield(vcpu)`:
1. If vcpu.arch.in_yield_loop: return.
2. vcpu.arch.in_yield_loop = true.
3. For each potential target vCPU: yield_to.
4. vcpu.arch.in_yield_loop = false.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mwait_in_guest_gating` | INVARIANT | per-VM mwait_in_guest decided at create; per-vCPU consistent. |
| `pause_window_bounded` | INVARIANT | PLE-window in [0, U32_MAX] sane range. |
| `yield_loop_no_recursion` | INVARIANT | per-vCPU in_yield_loop prevents recursive yield. |
| `halt_state_atomic` | INVARIANT | vCPU mp_state transitions atomic. |

### Layer 2: TLA+

`virt/kvm/mwait_pause_intercept.tla`:
- Per-vCPU instruction-state ∈ {Running, MwaitPending, Paused, Halted}.
- Properties:
  - `safety_mwait_only_halts_when_intercepted` — MWAIT-treated-as-HLT only when intercept active.
  - `safety_pause_yields_when_intercepted` — PAUSE-vmexit triggers yield_to_loop.
  - `liveness_halted_eventually_resumes` — Halted → Running on pending event.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuVmx::handle_mwait` post: vCPU halted OR resumed per kvm.mwait_in_guest | `VcpuVmx::handle_mwait` |
| `VcpuVmx::handle_pause` post: yield-to-loop dispatched OR resumed | `VcpuVmx::handle_pause` |
| `Vcpu::halt` post: vcpu.mp_state == HALTED OR userspace exit | `Vcpu::halt` |
| `Vcpu::yield_to` post: target vCPU scheduled if possible | `Vcpu::yield_to` |

### Layer 4: Verus/Creusot functional

`Per-MWAIT/PAUSE: KVM intercepts vs runs-natively decided by per-VM gate; vCPU yield-loop terminates` semantic equivalence: per-instruction the resulting vCPU state matches per-VM intercept policy.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

MWAIT/PAUSE-specific reinforcement:

- **KVM_X86_DISABLE_EXITS_MWAIT/_PAUSE/_HLT requires explicit opt-in** — defense against unauthorized CPU-monopolization.
- **Per-VM CPU-pinning + cgroup-cpuset recommended** when MWAIT-in-guest enabled — defense against host-CPU starvation.
- **PLE-window + PLE-gap tunable** — defense against pathological lock-spin patterns.
- **Per-vCPU yield_loop in_yield_loop flag** — defense against recursive yield.
- **HLT-in-guest gated per-VM** — defense against guest holding CPU indefinitely.
- **MWAIT-as-HLT default** — defense against unbounded native-MWAIT spin.
- **MONITOR-MSR pass-through gated** — defense against guest configuring monitored-addr in host kernel range.
- **KVM_FEATURE_PV_UNHALT preferred** — defense against MWAIT-vmexit-overhead by using paravirt yield.
- **Per-vCPU pause_count rate-limit log** — defense against spam-log on pathological PAUSE.
- **Live-migrate carries per-VM disable_exits flags** — defense against post-migrate inconsistent intercept policy.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — KVM_X86_DISABLE_EXITS payload bounded.
- **PAX_KERNEXEC** — PLE / mwait-intercept dispatch RO after init.
- **PAX_RANDKSTACK** — randomized kstack per KVM_RUN.
- **PAX_REFCOUNT** — pause/mwait counter saturating.
- **PAX_MEMORY_SANITIZE** — per-vCPU pause_count zeroed on vCPU destroy.
- **PAX_UDEREF** — disable_exits cap user pointer validated.
- **PAX_RAP / kCFI** — pause-handler / mwait-handler callbacks type-checked.
- **GRKERNSEC_HIDESYM** — monitored-addr / PAUSE-RIP redacted.
- **GRKERNSEC_DMESG** — PAUSE/MWAIT spam rate-limited.

MWAIT-specific:

- **CAP_SYS_ADMIN on KVM_X86_DISABLE_EXITS** — defense against unprivileged CPU-monopolization.
- **MWAIT/HLT/PAUSE intercept default-on** — disable requires explicit opt-in + cpuset pinning.
- **MONITOR address range-checked** — defense against guest monitoring host-kernel VA.

Rationale: MWAIT/HLT/PAUSE intercept disable lets a guest hog a host CPU; CAP+opt-in + RAP-checked dispatch ensure only privileged userspace can take that trade-off.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VMX core (covered in `x86-vmx.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- KVM PV-IPI / PV-yield (covered in `x86-pv-ipi.md` Tier-3)
- HLT/idle integration (covered in `x86-vmx.md` + `kvm-core.md` Tier-3s)
- AMD SVM intercept (covered in `x86-svm.md` Tier-3; analog)
- MTF / single-step (covered in `x86-mtf.md` Tier-3; distinct mechanism)
- Implementation code
