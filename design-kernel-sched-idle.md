---
title: "Tier-3: kernel/sched/idle.c — Idle task scheduling (per-CPU `swapper`)"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Per-CPU idle task (`swapper/N`) runs the **idle scheduling class** when no other task is runnable. Per-iter calls `cpuidle_idle_call()` → `cpuidle_select` picks per-CPU `cpuidle_state` based on per-governor (menu / teo / ladder); per-state enters per-arch low-power instruction (e.g., x86 MWAIT, ARM WFI). Per-state has (target_residency, exit_latency, power-cost). Per-NOHZ_FULL CPUs may avoid sched-tick. Per-do_idle loop balances per-PM cpufreq scaling. Critical for: per-CPU power-efficient idle, throughput-vs-latency tradeoff via per-state selection.

This Tier-3 covers `idle.c` (~590 lines).

### Acceptance Criteria

- [ ] AC-1: Boot completes; per-CPU swapper task runs.
- [ ] AC-2: rq.nr_running == 0: pick_next_task returns idle.
- [ ] AC-3: do_idle calls cpuidle_idle_call.
- [ ] AC-4: cpuidle_select picks state per residency.
- [ ] AC-5: MWAIT entered on x86 C-state.
- [ ] AC-6: NOHZ_IDLE: tick stopped if no pending timer.
- [ ] AC-7: IRQ arrives → need_resched → idle loop exits.
- [ ] AC-8: /sys/devices/system/cpu/cpu0/cpuidle/state0/time: residency accumulates.
- [ ] AC-9: cpuidle governor change: per-/sys/devices/system/cpu/cpuidle/current_governor.
- [ ] AC-10: CPU-offline: idle_task_exit called.

### Architecture

`Idle::cpu_idle_loop(cpu)`:
1. /* Per-CPU idle task entry */
2. for { Idle::do_idle(); }.

`Idle::do_idle()`:
1. while !need_resched():
   - rcu_idle_enter().
   - local_irq_disable().
   - tick_nohz_idle_enter().
   - if !need_resched(): Idle::cpuidle_idle_call().
   - else: local_irq_enable().
   - tick_nohz_idle_exit().
   - rcu_idle_exit().
2. schedule_idle().

`Idle::cpuidle_idle_call()`:
1. drv = cpuidle_get_cpu_driver(this_cpu).
2. dev = cpuidle_get_device().
3. if !drv ∨ !dev: Idle::default_idle_call(); return.
4. state_idx = cpuidle_governor.select(drv, dev, &stop_tick).
5. if stop_tick: tick_nohz_idle_stop_tick().
6. entered = cpuidle_enter(drv, dev, state_idx).
7. cpuidle_governor.reflect(dev, entered).

`Idle::default_idle_call()`:
1. /* Per-arch default */
2. cpu_idle_set_state(NULL).
3. arch_cpu_idle().  // e.g., HLT.
4. cpu_idle_clear_state().

`Idle::cpuidle_select(drv, dev, stop_tick) -> i32`:
1. /* Per-governor heuristic */
2. e.g. menu: predict-residency based on recent history.
3. return state_idx of best-fit state.

`Idle::cpuidle_enter(drv, dev, state_idx)`:
1. state = &drv.states[state_idx].
2. cpu_idle_set_state(state).
3. time_start = local_clock().
4. ret = state.enter(dev, drv, state_idx).
5. time_end = local_clock().
6. dev.last_residency_ns = time_end - time_start.
7. cpu_idle_clear_state().
8. return ret.

`Idle::play_idle(duration_us)` (for testing):
1. play_idle_precise(duration_us * NSEC_PER_USEC, latency_ns).

### Out of Scope

- kernel/sched/core (covered in `core.md` Tier-3)
- drivers/cpuidle/ (covered separately if expanded)
- Per-arch cpu_idle implementation (covered in `arch/x86/00-overview.md` separately)
- NOHZ_FULL tick subsystem (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `cpu_idle_loop()` (entry per-CPU) | per-CPU idle loop | `Idle::cpu_idle_loop` |
| `do_idle()` | per-iter | `Idle::do_idle` |
| `idle_task_exit()` | per-CPU offline | `Idle::task_exit` |
| `cpuidle_idle_call()` | per-PM dispatch | `Idle::cpuidle_idle_call` |
| `default_idle_call()` | per-fallback | `Idle::default_idle_call` |
| `play_idle()` | per-test inject | `Idle::play_idle` |
| `set_tsk_need_resched()` | per-wake-event | shared |
| `sched_idle_set_state()` | per-set | `Idle::set_state` |
| `tick_nohz_idle_enter()` / `_exit()` | per-NOHZ-idle tick | shared |
| `cpuidle_select()` | per-governor pick | `Idle::cpuidle_select` |
| `cpuidle_enter()` | per-state enter | `Idle::cpuidle_enter` |
| `idle_sched_class` | per-class ops | `IdleSchedClass` |

### compatibility contract

REQ-1: Per-CPU idle task:
- Per-CPU `swapper/N` task (PID 0 on boot CPU; per-CPU init at smp-bringup).
- Per-task->sched_class == &idle_sched_class.
- pick_next_task_idle returns idle task when rq.nr_running == 0.

REQ-2: do_idle:
- Loop:
  - rcu_idle_enter.
  - if need_resched: break.
  - Per-cpuidle_idle_call (with governor + state).
  - rcu_idle_exit.

REQ-3: cpuidle_idle_call:
- Pick state via cpuidle_governor.select.
- if cpuidle_state.flags & CPUIDLE_FLAG_TLB_FLUSHED: per-side-effects.
- cpuidle_enter(drv, state).

REQ-4: Per-cpuidle_state:
- target_residency_ns: expected idle time.
- exit_latency_ns: per-state exit cost.
- power_usage: per-state power draw.
- flags: TLB_FLUSHED / OFF / IDLE_DETECTED / etc.
- enter: per-state entry fn (e.g., mwait or wfi).

REQ-5: Per-x86 idle states:
- C0: running.
- C1: HLT.
- C1E: HLT with PM-tickless.
- C2 / C3 / C6 / C7 / C8 / C10: deeper via MWAIT.

REQ-6: Per-NOHZ_IDLE:
- tick_nohz_idle_enter: stop sched-tick if no pending timers.
- Per-resume: tick_nohz_idle_exit + re-arm.

REQ-7: Per-governor:
- menu: heuristic per-history-based pick.
- teo: Time-of-Event-Oriented (simpler, no history).
- ladder: per-state-step-up over time.
- haltpoll: KVM-guest poll-before-halt.

REQ-8: Per-arch fallback:
- default_idle_call: per-arch default_idle (HLT on x86, WFI on ARM).

REQ-9: idle_sched_class:
- pick_next_task = pick_next_task_idle.
- task_tick = task_tick_idle (no-op).
- prio_changed = no-op.

REQ-10: Per-PM-suspend:
- Per-CPU enters deepest state.

REQ-11: Per-stat:
- /sys/devices/system/cpu/cpuN/cpuidle/state*/{name, time, usage, latency, residency, ...}.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `idle_task_per_cpu_unique` | INVARIANT | per-CPU one swapper/N task. |
| `need_resched_exits_loop` | INVARIANT | per-do_idle: need_resched ⟹ loop exits. |
| `rcu_idle_balanced` | INVARIANT | per-iter rcu_idle_enter paired with rcu_idle_exit. |
| `cpuidle_state_lt_count` | INVARIANT | per-select state_idx < drv.state_count. |
| `irq_enabled_at_exit` | INVARIANT | per-do_idle exit: local IRQs enabled. |

### Layer 2: TLA+

`kernel/sched/idle.tla`:
- Per-CPU do_idle loop + cpuidle_select + state-enter + need_resched-exit.
- Properties:
  - `safety_no_runnable_other_pre_idle` — per-do_idle: rq.nr_running == 0.
  - `safety_need_resched_eventually_exits` — per-IRQ wake: need_resched ⟹ idle exits ≤ 1 iter.
  - `liveness_per_irq_resumes` — per-IRQ → tick_resume + schedule_idle.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Idle::do_idle` post: rcu_idle_enter/exit balanced; need_resched=true | `Idle::do_idle` |
| `Idle::cpuidle_idle_call` post: cpuidle_state entered | `Idle::cpuidle_idle_call` |
| `Idle::default_idle_call` post: arch_cpu_idle invoked | `Idle::default_idle_call` |

### Layer 4: Verus/Creusot functional

`Per-CPU idle task → cpuidle_governor.select → per-state enter via mwait/wfi → per-IRQ wake → exit` semantic equivalence: per-Documentation/admin-guide/pm/cpuidle.rst.

### hardening

(Inherits row-1 features from `kernel/sched/core.md` § Hardening.)

Idle reinforcement:

- **Per-need_resched checked atomic** — defense against per-spurious-wake-loop.
- **Per-state_idx bounded by drv.state_count** — defense against per-OOR state access.
- **Per-arch_cpu_idle default fallback** — defense against per-cpuidle-driver missing.
- **Per-NOHZ_FULL careful sched-tick stop** — defense against per-timer miss.
- **Per-governor reflect for adapt** — defense against per-state mispredict.
- **Per-rcu_idle_enter/exit balanced** — defense against per-RCU stuck-grace.
- **Per-cpu_idle_set_state for trace/stats** — defense against per-state-attrib mismatch.
- **Per-CPU-offline idle_task_exit synchronized** — defense against per-late-IRQ.
- **Per-play_idle test-only** — defense against per-prod idle-inject.
- **Per-sysfs CAP_SYS_ADMIN for governor change** — defense against unprivileged power-policy.

