# Tier-3: arch/x86/kvm/lapic.c (TSC-Deadline subset) — TSC-Deadline timer mode (per-vCPU MSR_IA32_TSC_DEADLINE + lapic_timer_advance)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-lapic.md
upstream-paths:
  - arch/x86/kvm/lapic.c (tsc_deadline sections; see grep "tsc_deadline\|TSCDEADLINE\|lapic_timer_advance")
  - arch/x86/kvm/x86.c (TSC-deadline MSR dispatch)
  - arch/x86/include/asm/apicdef.h (APIC_LVT_TIMER_TSCDEADLINE)
-->

## Summary

LAPIC TSC-Deadline mode (Intel-arch-perfmon-3+) is the modern timer mode where guest writes MSR_IA32_TSC_DEADLINE with target TSC value; CPU fires LVT_TIMER interrupt when TSC reaches the deadline. Replaces older periodic + one-shot modes via APIC.LVTTIMER_DCR. KVM virtualizes per-vCPU: stash deadline in apic.lapic_timer.tscdeadline; arm hrtimer at corresponding host-time; on hrtimer fire: inject LVT_TIMER vector. Per-vCPU lapic_timer_advance compensates for vmenter-overhead by firing hrtimer slightly early so guest sees timer at correct wall-time.

This Tier-3 covers TSC-deadline timer in `arch/x86/kvm/lapic.c` (~200-300 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_set_lapic_tscdeadline_msr(vcpu, data)` | per-vCPU TSC-deadline MSR write | `Lapic::set_tscdeadline_msr` |
| `kvm_get_lapic_tscdeadline_msr(vcpu)` | per-vCPU read | `Lapic::get_tscdeadline_msr` |
| `apic.lapic_timer.tscdeadline` | per-vCPU deadline shadow | `LapicTimer::tscdeadline` |
| `apic.lapic_timer.expired_tscdeadline` | per-vCPU last-expired-deadline | `LapicTimer::expired_tscdeadline` |
| `apic.lapic_timer.timer_advance_ns` | per-vCPU early-fire ns | `LapicTimer::timer_advance_ns` |
| `apic_set_tsc_deadline(apic, tscdeadline)` | per-vCPU set | `Lapic::apic_set_tsc_deadline` |
| `start_sw_tscdeadline(apic)` | per-vCPU arm hrtimer | `Lapic::start_sw_tscdeadline` |
| `apic_timer_expired(apic, fromtimer)` | per-vCPU expire callback | `Lapic::apic_timer_expired` |
| `__wait_lapic_expire(vcpu, ns)` | per-vCPU spin-wait | `Lapic::__wait_lapic_expire` |
| `wait_lapic_expire(vcpu)` | wrapper | `Lapic::wait_lapic_expire` |
| `adjust_lapic_timer_advance(vcpu, advance_expire_delta)` | per-vCPU dynamic adjust | `Lapic::adjust_lapic_timer_advance` |
| `apic_lvtt_tscdeadline(apic)` | per-vCPU mode query | `Lapic::lvtt_tscdeadline` |
| `MSR_IA32_TSC_DEADLINE` (0x6E0) | guest-visible MSR | UAPI |

## Compatibility contract

REQ-1: Per-vCPU LAPIC timer modes (LVT_TIMER bit[18:17]):
- 0 = One-shot.
- 1 = Periodic.
- 2 = TSC-Deadline.

REQ-2: TSC-Deadline activation:
- LVT_TIMER bits[18:17] = 0b10.
- Subsequent WRMSR(MSR_IA32_TSC_DEADLINE) sets deadline.

REQ-3: TSC-Deadline semantics:
- Guest writes future TSC value.
- CPU fires LVT_TIMER interrupt when guest TSC ≥ deadline.
- Subsequent WRMSR resets timer to new deadline.
- WRMSR(0) disables timer.

REQ-4: Per-vCPU deadline arming:
1. lapic_timer.tscdeadline := guest deadline.
2. host_deadline := vcpu_tsc_to_host_ns(deadline) - timer_advance_ns.
3. hrtimer_start(&apic.lapic_timer.timer, host_deadline, HRTIMER_MODE_ABS_PINNED).

REQ-5: Per-vCPU `timer_advance_ns`:
- Early-fire compensation: hrtimer fires this many ns before guest deadline.
- Allows for vmenter overhead so guest TSC = deadline at user-visible time.
- Default: 1000 ns.
- Adaptively tuned via adjust_lapic_timer_advance.

REQ-6: Per-vCPU spin-wait:
- After hrtimer fires + before vmenter: KVM calls __wait_lapic_expire.
- Spin-loop until guest TSC ≥ deadline.
- Reduces tail-latency by absorbing vmenter overhead.

REQ-7: Per-vCPU adaptive timer-advance:
- adjust_lapic_timer_advance(vcpu, advance_expire_delta):
  - delta = guest_tsc - tscdeadline.
  - If delta > threshold: timer_advance_ns -= step.
  - If delta < -threshold: timer_advance_ns += step.
- Self-tunes timer-advance for minimum wait.

REQ-8: Per-vCPU `apic_timer_expired` flow:
1. apic.lapic_timer.expired_tscdeadline := tscdeadline.
2. apic_timer_expired_inject = 1.
3. kvm_make_request(KVM_REQ_PENDING_TIMER, vcpu).
4. kvm_vcpu_kick.

REQ-9: APICv + TSC-deadline:
- With APICv-TPR-shadow + APICv-virt-int-delivery: per-LVT_TIMER injection direct to vAPIC IRR.
- No vmexit on per-injection.

REQ-10: Live migration:
- Per-vCPU tscdeadline + timer-mode migrated.
- Post-migrate: re-arm hrtimer using new host-CPU TSC.

REQ-11: Per-vCPU hrtimer cleanup:
- vCPU destroy: hrtimer_cancel(apic.lapic_timer.timer).
- Avoid post-destroy fire causing UAF.

REQ-12: TSC-scaling integration:
- Per-vCPU TSC-scaling-ratio applied: host_deadline = (guest_deadline - guest_tsc_at_now) / scaling + now_host_tsc.
- Then host_deadline → host_ns via TSC-to-ns.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest with TSC-deadline (default modern Linux): kernel uses TSC-deadline.
- [ ] AC-2: Guest writes MSR_IA32_TSC_DEADLINE = TSC + 100us: hrtimer armed; LVT_TIMER fires ~100us later.
- [ ] AC-3: Disable: WRMSR(MSR_IA32_TSC_DEADLINE, 0); hrtimer cancelled.
- [ ] AC-4: timer_advance: per-vCPU early-fire reduces wait; counter visible.
- [ ] AC-5: Adaptive tuning: per-vCPU timer_advance_ns self-adjusts based on observed delta.
- [ ] AC-6: APICv direct injection: per-fire TSC-deadline IRQ delivered without vmexit.
- [ ] AC-7: Live migration: pending TSC-deadline preserved.
- [ ] AC-8: Multi-vCPU: per-vCPU deadline distinct.
- [ ] AC-9: kvm-unit-tests `tsc_deadline_timer` test passes.
- [ ] AC-10: Performance: TSC-deadline IRQ-latency within 1us of bare-metal.

## Architecture

`Lapic::set_tscdeadline_msr(vcpu, data)`:
1. apic := vcpu.arch.apic.
2. If !apic_lvtt_tscdeadline(apic): return -EINVAL.
3. apic.lapic_timer.tscdeadline = data.
4. If data == 0:
   - hrtimer_cancel(&apic.lapic_timer.timer).
5. Else:
   - start_sw_tscdeadline(apic).
6. Return 0.

`Lapic::start_sw_tscdeadline(apic)`:
1. now_tsc := rdtsc_ordered().
2. If apic.lapic_timer.tscdeadline <= now_tsc:
   - apic_timer_expired(apic, true).
   - return.
3. host_deadline_ns := (apic.lapic_timer.tscdeadline - now_tsc) * tsc_to_ns - apic.lapic_timer.timer_advance_ns.
4. host_deadline_abs := ktime_get() + host_deadline_ns.
5. hrtimer_start(&apic.lapic_timer.timer, host_deadline_abs, HRTIMER_MODE_ABS_HARD).

`Lapic::apic_timer_expired(apic, fromtimer)`:
1. If fromtimer:
   - apic.lapic_timer.expired_tscdeadline = apic.lapic_timer.tscdeadline.
2. apic_lvt_emit_lvt_timer(apic).
3. kvm_make_request(KVM_REQ_PENDING_TIMER, vcpu).
4. kvm_vcpu_kick(vcpu).

`Lapic::wait_lapic_expire(vcpu)`:
1. apic := vcpu.arch.apic.
2. If !apic_lvtt_tscdeadline(apic): return.
3. tscdeadline := apic.lapic_timer.expired_tscdeadline.
4. guest_tsc := rdtsc_ordered() + vcpu.arch.tsc_offset.
5. delta := guest_tsc - tscdeadline.
6. adjust_lapic_timer_advance(vcpu, delta).
7. If guest_tsc < tscdeadline:
   - __wait_lapic_expire(vcpu, tscdeadline - guest_tsc).

`Lapic::adjust_lapic_timer_advance(vcpu, delta)`:
1. apic := vcpu.arch.apic.
2. If delta > threshold (e.g., 8000ns):
   - apic.lapic_timer.timer_advance_ns += step (e.g., 200ns).
3. Else if delta < -threshold:
   - apic.lapic_timer.timer_advance_ns = max(0, timer_advance_ns - step).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tscdeadline_no_overflow` | INVARIANT | per-vCPU tscdeadline u64 wrap-safe arithmetic. |
| `hrtimer_arm_idempotent` | INVARIANT | per-set_tscdeadline cancels old + arms new. |
| `timer_advance_bounded` | INVARIANT | per-vCPU timer_advance_ns ≥ 0; capped at sane upper bound. |
| `lvt_mode_tscdeadline_active` | INVARIANT | per-vCPU only set when LVT_TIMER mode == TSC_DEADLINE. |
| `wait_lapic_bounded` | INVARIANT | __wait_lapic_expire spin-loop bounded by max-wait threshold. |

### Layer 2: TLA+

`virt/kvm/tsc_deadline_timer.tla`:
- Per-vCPU timer state ∈ {Disabled, Armed(deadline), Fired, Cancelled}.
- Properties:
  - `safety_no_fire_before_deadline` — Fired implies guest_tsc ≥ deadline.
  - `safety_set_resets_old_arm` — Armed(d1) → Armed(d2) cancels d1's hrtimer.
  - `liveness_armed_eventually_fires` — assuming TSC advances, Armed(d) eventually Fired.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Lapic::set_tscdeadline_msr` post: per-vCPU tscdeadline + hrtimer state consistent | `Lapic::set_tscdeadline_msr` |
| `Lapic::start_sw_tscdeadline` post: hrtimer armed at host_deadline | `Lapic::start_sw_tscdeadline` |
| `Lapic::apic_timer_expired` post: KVM_REQ_PENDING_TIMER set; vCPU kicked | `Lapic::apic_timer_expired` |
| `Lapic::wait_lapic_expire` post: guest_tsc ≥ deadline before vmenter | `Lapic::wait_lapic_expire` |

### Layer 4: Verus/Creusot functional

`Per-vCPU MSR_IA32_TSC_DEADLINE write: hrtimer arms host clock to fire LVT_TIMER at guest TSC = deadline` semantic equivalence: per-deadline the LVT_TIMER fires when guest sees TSC reach the value (within timer-advance precision).

## Hardening

(Inherits row-1 features from `virt/kvm/x86-lapic.md` § Hardening.)

TSC-deadline-specific reinforcement:

- **Per-vCPU tscdeadline arithmetic wrap-safe** — defense against u64-arithmetic-overflow.
- **timer_advance_ns capped** — defense against pathological self-tuning.
- **__wait_lapic_expire bounded** — defense against guest-controlled deadline causing host CPU stall.
- **Per-vCPU hrtimer cancelled at vCPU-destroy** — defense against post-destroy fire UAF.
- **TSC-scaling applied per-VM** — defense against per-vCPU TSC drift causing wrong deadline.
- **APICv direct-inject coordinated** — defense against direct-inject with stale TPR.
- **Live-migrate timer-state serialized** — defense against post-migrate stuck-armed timer.
- **Per-LVT_TIMER mode validated** — defense against MSR_IA32_TSC_DEADLINE write in wrong mode.
- **Per-vCPU rdtsc_ordered() barriers** — defense against speculative RDTSC reorder.
- **adjust_lapic_timer_advance per-vCPU** — defense against cross-vCPU advance interference.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- LAPIC core (covered in `x86-lapic.md` Tier-3)
- TSC virtualization (covered in `x86-rdtsc.md` Tier-3)
- Periodic / one-shot APIC timer modes (covered in `x86-lapic.md` Tier-3)
- APICv (covered in `x86-vmx-apicv.md` Tier-3)
- AVIC (covered in `x86-svm-avic.md` Tier-3)
- Implementation code
