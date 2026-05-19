# Tier-3: kernel/power/suspend.c — System suspend (S2I / S2RAM / S2D)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/power/00-overview.md
upstream-paths:
  - kernel/power/suspend.c (~649 lines)
  - include/linux/suspend.h (suspend_state_t, platform_suspend_ops, platform_s2idle_ops, pm_suspend_target_state)
  - kernel/power/main.c (sysfs /sys/power/state, mem_sleep)
  - kernel/power/process.c (freeze_processes, thaw_processes)
  - drivers/base/power/main.c (dpm_suspend_start, _late, _noirq, dpm_resume_*)
  - kernel/cpu.c (pm_sleep_disable_secondary_cpus, _enable_secondary_cpus)
  - kernel/power/wakeup_reason.c (pm_wakeup_pending, wakeup-source detection)
  - drivers/base/syscore.c (syscore_suspend, syscore_resume)
-->

## Summary

`kernel/power/suspend.c` implements the **system-wide suspend pipeline** for the four `suspend_state_t` values: `PM_SUSPEND_TO_IDLE` (S2I, "freeze"), `PM_SUSPEND_STANDBY` (S1, "standby"), `PM_SUSPEND_MEM` (S3, "deep" / suspend-to-RAM), and is the entry point for hibernation-adjacent low-power transitions. The externally visible `pm_suspend(state)` validates the state against `valid_state()` + sysfs `/sys/power/state` registration, then calls `enter_state()` which takes `system_transition_mutex`, optionally syncs filesystems, runs the **suspend prepare** phase (notifiers → freeze userspace processes → console-suspend), runs **suspend devices and enter** (`platform_suspend_begin` → `dpm_suspend_start(PMSG_SUSPEND)` → loop of `suspend_enter`), and on resume mirrors the unwind. `suspend_enter` walks the four device-PM phases (`dpm_suspend_late` → `dpm_suspend_noirq`), disables secondary CPUs (`pm_sleep_disable_secondary_cpus`), disables IRQs via `arch_suspend_disable_irqs`, freezes the timekeeping core (`syscore_suspend`), and finally calls `suspend_ops->enter(state)` for S1/S3 or `s2idle_loop()` for S2I. Wakeup-event detection (`pm_wakeup_pending()`) is polled at every phase boundary so a wakeup arriving during prepare causes immediate abort. Critical for: laptop lid-close suspend, smartphone S2I, server suspend-on-idle, and as a building block for hibernate (S4) which uses many of the same dpm_* phases.

This Tier-3 covers `kernel/power/suspend.c` (~649 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enum { PM_SUSPEND_ON, _TO_IDLE, _STANDBY, _MEM, _MAX }` | per-state enum | `SuspendState` |
| `struct platform_suspend_ops` | per-platform vtable | `PlatformSuspendOps` |
| `struct platform_s2idle_ops` | per-S2I vtable | `PlatformS2idleOps` |
| `pm_suspend()` | per-entry dispatcher | `Suspend::pm_suspend` |
| `enter_state()` | per-state common entry | `Suspend::enter_state` |
| `suspend_prepare()` | per-pre-suspend prep | `Suspend::prepare` |
| `suspend_devices_and_enter()` | per-device-suspend + enter | `Suspend::devices_and_enter` |
| `suspend_enter()` | per-late/noirq/syscore enter | `Suspend::enter` |
| `suspend_finish()` | per-thaw + console restore | `Suspend::finish` |
| `valid_state()` | per-state-supported predicate | `Suspend::valid_state` |
| `sleep_state_supported()` | per-state + CXL check | `Suspend::sleep_state_supported` |
| `platform_suspend_begin()` | per-ops.begin shim | `Suspend::platform_begin` |
| `platform_suspend_prepare()` | per-ops.prepare shim | `Suspend::platform_prepare` |
| `platform_suspend_prepare_late()` | per-s2idle.prepare shim | `Suspend::platform_prepare_late` |
| `platform_suspend_prepare_noirq()` | per-ops.prepare_late shim | `Suspend::platform_prepare_noirq` |
| `platform_resume_noirq()` | per-resume noirq shim | `Suspend::platform_resume_noirq` |
| `platform_resume_early()` | per-s2idle.restore | `Suspend::platform_resume_early` |
| `platform_resume_finish()` | per-ops.finish shim | `Suspend::platform_resume_finish` |
| `platform_resume_end()` | per-ops.end / s2idle.end | `Suspend::platform_resume_end` |
| `platform_recover()` | per-ops.recover on failure | `Suspend::platform_recover` |
| `platform_suspend_again()` | per-ops.suspend_again loop | `Suspend::platform_suspend_again` |
| `s2idle_loop()` | per-S2I idle-poll loop | `S2Idle::loop` |
| `s2idle_enter()` | per-S2I idle wait + wake | `S2Idle::enter` |
| `s2idle_wake()` | per-S2I wake-event | `S2Idle::wake` |
| `s2idle_begin()` | per-S2I init | `S2Idle::begin` |
| `s2idle_set_ops()` | per-S2I vtable register | `S2Idle::set_ops` |
| `suspend_set_ops()` | per-platform vtable register | `Suspend::set_ops` |
| `suspend_valid_only_mem()` | per-helper for mem-only platforms | `Suspend::valid_only_mem` |
| `arch_suspend_disable_irqs()` | per-arch IRQ disable | `Arch::suspend_disable_irqs` |
| `arch_suspend_enable_irqs()` | per-arch IRQ enable | `Arch::suspend_enable_irqs` |
| `pm_suspend_default_s2idle()` | per-mem_sleep default check | `Suspend::default_s2idle` |
| `pm_states[]` / `mem_sleep_states[]` | per-state name tables | `Suspend::states_table` |
| `pm_suspend_target_state` | per-active state global | `Suspend::target_state` |
| `pm_suspend_global_flags` | per-FW_SUSPEND/FW_RESUME/NO_PLATFORM bits | `Suspend::global_flags` |

## Compatibility contract

REQ-1: suspend_state_t enumeration (per-include/linux/suspend.h):
- PM_SUSPEND_ON = 0 — not suspended (running).
- PM_SUSPEND_TO_IDLE = 1 — S0ix / "freeze": devices suspended, CPUs idle, no platform sleep.
- PM_SUSPEND_STANDBY = 2 — S1 / "standby": power-on suspend, CPU caches flushed, low-power state retained by platform.
- PM_SUSPEND_MEM = 3 — S3 / "deep" / suspend-to-RAM: most of the system powered off, RAM in self-refresh.
- PM_SUSPEND_MIN = PM_SUSPEND_TO_IDLE.
- PM_SUSPEND_MAX = 4 — sentinel.

REQ-2: pm_suspend(state) — exported entry:
- if state ≤ PM_SUSPEND_ON ∨ state ≥ PM_SUSPEND_MAX: return -EINVAL.
- pr_info("suspend entry (%s)", mem_sleep_labels[state]).
- error = enter_state(state).
- dpm_save_errno(error) — per-stat sysfs.
- pr_info("suspend exit").
- return error.

REQ-3: enter_state(state):
- trace_suspend_resume(TPS("suspend_enter"), state, true).
- if state == PM_SUSPEND_TO_IDLE ∧ CONFIG_PM_DEBUG: reject if pm_test_level ∈ {TEST_FREEZER, TEST_DEVICES, TEST_PLATFORM, TEST_CPUS} doesn't include s2idle-compatible levels (only none/freezer/devices/platform allowed).
- else if !valid_state(state): return -EINVAL.
- if !mutex_trylock(&system_transition_mutex): return -EBUSY (concurrent sleep attempt).
- if state == PM_SUSPEND_TO_IDLE: s2idle_begin() — s2idle_state = S2IDLE_STATE_NONE.
- if sync_on_suspend_enabled: pm_sleep_fs_sync() — sync dirty fs; on fail goto Unlock.
- pm_suspend_clear_flags() — clear FW_SUSPEND / FW_RESUME / NO_PLATFORM bits.
- error = suspend_prepare(state); on fail goto Unlock.
- if suspend_test(TEST_FREEZER): goto Finish (debug test mode).
- error = suspend_devices_and_enter(state).
- Finish: events_check_enabled = false; suspend_finish().
- Unlock: mutex_unlock(&system_transition_mutex); return error.

REQ-4: suspend_prepare(state):
- if !sleep_state_supported(state): return -EPERM.
- pm_prepare_console() — switch to suspend console.
- pm_notifier_call_chain_robust(PM_SUSPEND_PREPARE, PM_POST_SUSPEND) — runs registered PM notifiers; pairing call-chain restores on error.
- filesystems_freeze(filesystem_freeze_enabled) — quiesce fs.
- trace_suspend_resume(TPS("freeze_processes"), 0, true).
- suspend_freeze_processes() — freeze userspace + freezable kthreads (sends FROZEN flag; waits for all tasks to enter refrigerator).
- trace_suspend_resume(TPS("freeze_processes"), 0, false).
- On freeze failure: dpm_save_failed_step(SUSPEND_FREEZE); filesystems_thaw(); pm_notifier_call_chain(PM_POST_SUSPEND); pm_restore_console().

REQ-5: suspend_devices_and_enter(state):
- if !sleep_state_supported(state): return -ENOSYS.
- pm_suspend_target_state = state.
- if state == PM_SUSPEND_TO_IDLE: pm_set_suspend_no_platform() (sets PM_SUSPEND_FLAG_NO_PLATFORM).
- platform_suspend_begin(state) — calls s2idle_ops->begin (S2I) or suspend_ops->begin(state).
- console_suspend_all() — quiesce all console drivers.
- suspend_test_start() — instrumentation marker.
- dpm_suspend_start(PMSG_SUSPEND) — runs full device-tree PM walk: prepare-phase then suspend-phase across all devices in dpm_list order; respects suspend-order dependency (parents after children).
- on dpm_suspend_start failure: pr_err + goto Recover_platform.
- suspend_test_finish("suspend devices").
- if suspend_test(TEST_DEVICES): goto Recover_platform.
- do { error = suspend_enter(state, &wakeup); } while (!error ∧ !wakeup ∧ platform_suspend_again(state)).
- Resume_devices: dpm_resume_end(PMSG_RESUME); console_resume_all().
- Close: platform_resume_end(state); pm_suspend_target_state = PM_SUSPEND_ON; return error.
- Recover_platform: platform_recover(state); goto Resume_devices.

REQ-6: suspend_enter(state, *wakeup):
- platform_suspend_prepare(state) — suspend_ops->prepare() for !S2I.
- dpm_suspend_late(PMSG_SUSPEND) — late-phase callback on every device.
- on fail: pr_err("late suspend of devices failed"); goto Platform_finish.
- platform_suspend_prepare_late(state) — s2idle_ops->prepare() for S2I.
- on fail: goto Devices_early_resume.
- dpm_suspend_noirq(PMSG_SUSPEND) — noirq-phase callback (IRQs still on but device IRQ handler must not race).
- on fail: pr_err("noirq suspend of devices failed"); goto Platform_early_resume.
- platform_suspend_prepare_noirq(state) — s2idle_ops->prepare_late() for S2I or suspend_ops->prepare_late() otherwise.
- on fail: goto Platform_wake.
- if suspend_test(TEST_PLATFORM): goto Platform_wake (debug).
- /* S2I branch */
- if state == PM_SUSPEND_TO_IDLE: s2idle_loop(); goto Platform_wake.
- /* S1/S3 branch */
- pm_sleep_disable_secondary_cpus() — offline all CPUs except boot CPU.
- on fail ∨ suspend_test(TEST_CPUS): goto Enable_cpus.
- arch_suspend_disable_irqs() — local_irq_disable on boot CPU (arch may also freeze TSC, save APIC state).
- BUG_ON(!irqs_disabled()).
- system_state = SYSTEM_SUSPEND.
- syscore_suspend() — calls every registered syscore_ops->suspend (timekeeping, clocksource, clockevents, irqchip-state, ...).
- if !error:
  - *wakeup = pm_wakeup_pending() — last-chance wakeup check.
  - if !(suspend_test(TEST_CORE) || *wakeup):
    - trace_suspend_resume(TPS("machine_suspend"), state, true).
    - error = suspend_ops->enter(state) — platform-specific final low-power transition (e.g. ACPI _S3 / mwait / wfi).
    - trace_suspend_resume(TPS("machine_suspend"), state, false).
  - else if *wakeup: error = -EBUSY.
  - syscore_resume().
- system_state = SYSTEM_RUNNING.
- arch_suspend_enable_irqs(); BUG_ON(irqs_disabled()).
- Enable_cpus: pm_sleep_enable_secondary_cpus().
- Platform_wake: platform_resume_noirq(state); dpm_resume_noirq(PMSG_RESUME).
- Platform_early_resume: platform_resume_early(state).
- Devices_early_resume: dpm_resume_early(PMSG_RESUME).
- Platform_finish: platform_resume_finish(state); return error.

REQ-7: s2idle_loop():
- pm_pr_dbg("suspend-to-idle").
- for (;;):
  - if s2idle_ops ∧ s2idle_ops->wake: if s2idle_ops->wake(): break.
  - else if pm_wakeup_pending(): break.
  - if s2idle_ops ∧ s2idle_ops->check: s2idle_ops->check().
  - s2idle_enter().
- pm_pr_dbg("resume from suspend-to-idle").

REQ-8: s2idle_enter():
- trace_suspend_resume(TPS("machine_suspend"), PM_SUSPEND_TO_IDLE, true).
- raw_spin_lock_irq(&s2idle_lock).
- if pm_wakeup_pending(): goto out (skip enter; reflect that wakeup arrived between check and lock).
- s2idle_state = S2IDLE_STATE_ENTER.
- raw_spin_unlock_irq.
- wake_up_all_idle_cpus() — kick each CPU into the idle loop.
- swait_event_exclusive(s2idle_wait_head, s2idle_state == S2IDLE_STATE_WAKE) — current CPU blocks until s2idle_wake().
- wake_up_all_idle_cpus() — wake all CPUs so they resume their timers + consistent state.
- raw_spin_lock_irq(&s2idle_lock).
- out: s2idle_state = S2IDLE_STATE_NONE.
- raw_spin_unlock_irq.
- trace_suspend_resume(TPS("machine_suspend"), PM_SUSPEND_TO_IDLE, false).

REQ-9: s2idle_wake() — called from wake-source IRQ context:
- raw_spin_lock_irqsave(&s2idle_lock).
- if s2idle_state > S2IDLE_STATE_NONE: s2idle_state = S2IDLE_STATE_WAKE; swake_up_one(&s2idle_wait_head).
- raw_spin_unlock_irqrestore.

REQ-10: valid_state(state):
- return suspend_ops ∧ suspend_ops->valid ∧ suspend_ops->valid(state) ∧ suspend_ops->enter.

REQ-11: sleep_state_supported(state):
- return state == PM_SUSPEND_TO_IDLE ∨ (valid_state(state) ∧ !cxl_mem_active()) — CXL memory currently incompatible with S1/S3.

REQ-12: suspend_set_ops(ops) — platform driver registration:
- sleep_flags = lock_system_sleep().
- suspend_ops = ops.
- if valid_state(PM_SUSPEND_STANDBY): mem_sleep_states[PM_SUSPEND_STANDBY] = "shallow"; pm_states[PM_SUSPEND_STANDBY] = "standby"; if mem_sleep_default == STANDBY: mem_sleep_current = STANDBY.
- if valid_state(PM_SUSPEND_MEM): mem_sleep_states[PM_SUSPEND_MEM] = "deep"; if mem_sleep_default >= MEM: mem_sleep_current = MEM.
- unlock_system_sleep(sleep_flags).

REQ-13: s2idle_set_ops(ops):
- sleep_flags = lock_system_sleep(); s2idle_ops = ops; unlock_system_sleep(sleep_flags).

REQ-14: platform_* shims (per-state branching):
- platform_suspend_prepare(state): state != S2I ∧ ops->prepare → ops->prepare().
- platform_suspend_prepare_late(state): state == S2I ∧ s2idle_ops->prepare → s2idle_ops->prepare().
- platform_suspend_prepare_noirq(state): if S2I → s2idle_ops->prepare_late else suspend_ops->prepare_late.
- platform_resume_noirq(state): if S2I → s2idle_ops->restore_early else suspend_ops->wake.
- platform_resume_early(state): if S2I ∧ s2idle_ops->restore → s2idle_ops->restore().
- platform_resume_finish(state): state != S2I ∧ ops->finish → ops->finish().
- platform_suspend_begin(state): if S2I ∧ s2idle_ops->begin → s2idle_ops->begin(); else if ops->begin → ops->begin(state); else 0.
- platform_resume_end(state): if S2I ∧ s2idle_ops->end → s2idle_ops->end(); else if ops->end → ops->end().
- platform_recover(state): state != S2I ∧ ops->recover → ops->recover().
- platform_suspend_again(state): state != S2I ∧ ops->suspend_again → ops->suspend_again().

REQ-15: arch_suspend_disable_irqs / _enable_irqs (weak default):
- Default: local_irq_disable / local_irq_enable.
- Arch override may also: freeze TSC, save+restore APIC/LAPIC, save MSR state, mark wakeup-from-suspend GPE, etc.

REQ-16: pm_states_init() (per-__init):
- pm_states[PM_SUSPEND_MEM] = "mem".
- pm_states[PM_SUSPEND_TO_IDLE] = "freeze".
- mem_sleep_states[PM_SUSPEND_TO_IDLE] = "s2idle".

REQ-17: mem_sleep_default kernel param:
- __setup("mem_sleep_default=", mem_sleep_default_setup).
- Accepts {"s2idle", "shallow", "deep"} → sets mem_sleep_default + mem_sleep_current.

REQ-18: pm_test_delay module param + suspend_test(level):
- CONFIG_PM_DEBUG only.
- If pm_test_level == level: print "Waiting %u second(s)"; busy-wait pm_test_delay seconds (mdelay/msleep depending on level) or until pm_wakeup_pending; return 1.

REQ-19: pm_suspend_global_flags — bit semantics (per-include/linux/suspend.h):
- PM_SUSPEND_FLAG_FW_SUSPEND = BIT(0) — firmware will perform suspend.
- PM_SUSPEND_FLAG_FW_RESUME = BIT(1) — firmware will perform resume.
- PM_SUSPEND_FLAG_NO_PLATFORM = BIT(2) — platform code skipped (S2I path).

REQ-20: pm_suspend_target_state — exported global tracking the active suspend_state_t (PM_SUSPEND_ON when idle).

## Acceptance Criteria

- [ ] AC-1: `echo mem > /sys/power/state`: pm_suspend(PM_SUSPEND_MEM) invoked; on platform without S3 support: returns -EINVAL.
- [ ] AC-2: `echo freeze > /sys/power/state`: enters s2idle_loop; CPUs idle; arch_suspend_disable_irqs NOT called for S2I.
- [ ] AC-3: pm_suspend(PM_SUSPEND_ON): returns -EINVAL.
- [ ] AC-4: pm_suspend(PM_SUSPEND_MAX): returns -EINVAL.
- [ ] AC-5: Concurrent pm_suspend on second CPU while first holds system_transition_mutex: second returns -EBUSY.
- [ ] AC-6: suspend_freeze_processes failure (e.g. a task ignored SIGKILL/freeze): suspend aborts; filesystems_thaw + pm_restore_console called; pm_suspend returns -EBUSY.
- [ ] AC-7: dpm_suspend_start failure: platform_recover invoked; resumed device tree consistent; pm_suspend returns negative errno.
- [ ] AC-8: pm_wakeup_pending() at every phase boundary aborts: prepare → -EBUSY; late-suspend → unwind; noirq → unwind; after syscore_suspend → -EBUSY with full unwind.
- [ ] AC-9: After successful S3 enter, on resume: syscore_resume → dpm_resume_noirq → dpm_resume_early → dpm_resume_end → console_resume_all → suspend_finish (thaw + filesystems_thaw + PM_POST_SUSPEND notifier + pm_restore_console).
- [ ] AC-10: suspend_ops->suspend_again returning true loops back into suspend_enter without resuming devices.
- [ ] AC-11: S2I wakeup via wake-source IRQ → s2idle_wake() → s2idle_enter swait_event releases → loop exits cleanly.
- [ ] AC-12: suspend_set_ops(NULL) and no S2I path: only PM_SUSPEND_TO_IDLE state available; "mem"/"standby" rejected at valid_state.

## Architecture

```
enum SuspendState {
  Pm_Suspend_On       = 0,
  Pm_Suspend_To_Idle  = 1,
  Pm_Suspend_Standby  = 2,
  Pm_Suspend_Mem      = 3,
  Pm_Suspend_Max      = 4,
}

struct PlatformSuspendOps {
  valid:          Option<fn(SuspendState) -> i32>,
  begin:          Option<fn(SuspendState) -> i32>,
  prepare:        Option<fn() -> i32>,
  prepare_late:   Option<fn() -> i32>,
  enter:          fn(SuspendState) -> i32,           // mandatory
  wake:           Option<fn()>,
  finish:         Option<fn()>,
  suspend_again:  Option<fn() -> bool>,
  end:            Option<fn()>,
  recover:        Option<fn()>,
}

struct PlatformS2idleOps {
  begin:          Option<fn() -> i32>,
  prepare:        Option<fn() -> i32>,
  prepare_late:   Option<fn() -> i32>,
  check:          Option<fn()>,
  wake:           Option<fn() -> bool>,
  restore_early:  Option<fn()>,
  restore:        Option<fn()>,
  end:            Option<fn()>,
}

static SUSPEND_OPS:       Atomic<Option<*PlatformSuspendOps>> = Atomic::new(None);
static S2IDLE_OPS:        Atomic<Option<*PlatformS2idleOps>>  = Atomic::new(None);
static SYSTEM_TRANSITION: Mutex<()>                            = Mutex::new(());
static S2IDLE_LOCK:       RawSpinLock                          = RawSpinLock::new();
static S2IDLE_WAIT_HEAD:  SwaitQueue                           = SwaitQueue::new();
```

`Suspend::pm_suspend(state) -> Result<()>`:
1. if state ≤ Pm_Suspend_On ∨ state ≥ Pm_Suspend_Max: return Err(EINVAL).
2. pr_info("suspend entry (%s)", mem_sleep_labels[state]).
3. let err = Suspend::enter_state(state).
4. dpm_save_errno(err).
5. pr_info("suspend exit").
6. err.

`Suspend::enter_state(state) -> Result<()>`:
1. trace_suspend_resume("suspend_enter", state, true).
2. if state == Pm_Suspend_To_Idle: validate pm_test_level compat (debug only).
3. else if !Suspend::valid_state(state): return Err(EINVAL).
4. let _guard = SYSTEM_TRANSITION.try_lock().ok_or(Err(EBUSY))?.
5. if state == Pm_Suspend_To_Idle: S2Idle::begin().
6. if sync_on_suspend_enabled: pm_sleep_fs_sync()?.
7. pm_suspend_clear_flags().
8. Suspend::prepare(state)?.
9. if suspend_test(TEST_FREEZER): goto Finish.
10. Suspend::devices_and_enter(state).
11. Finish: events_check_enabled = false; Suspend::finish().
12. Ok(()) or carry up the inner Err.

`Suspend::prepare(state) -> Result<()>`:
1. if !sleep_state_supported(state): return Err(EPERM).
2. pm_prepare_console().
3. pm_notifier_call_chain_robust(PM_SUSPEND_PREPARE, PM_POST_SUSPEND)?.
4. filesystems_freeze(filesystem_freeze_enabled).
5. trace_suspend_resume("freeze_processes", 0, true).
6. let err = suspend_freeze_processes().
7. trace_suspend_resume("freeze_processes", 0, false).
8. if Ok: return Ok(()).
9. dpm_save_failed_step(SUSPEND_FREEZE); filesystems_thaw(); pm_notifier_call_chain(PM_POST_SUSPEND); pm_restore_console().
10. Err(err).

`Suspend::devices_and_enter(state) -> Result<()>`:
1. if !sleep_state_supported(state): return Err(ENOSYS).
2. pm_suspend_target_state = state.
3. if state == Pm_Suspend_To_Idle: pm_set_suspend_no_platform().
4. Suspend::platform_begin(state)?.
5. console_suspend_all(); suspend_test_start().
6. dpm_suspend_start(PMSG_SUSPEND).
7. on Err: goto Recover_platform.
8. suspend_test_finish("suspend devices").
9. if suspend_test(TEST_DEVICES): goto Recover_platform.
10. let mut wakeup = false.
11. loop { let err = Suspend::enter(state, &mut wakeup); if err.is_err() || wakeup || !Suspend::platform_again(state) break; }.
12. Resume_devices: dpm_resume_end(PMSG_RESUME); console_resume_all().
13. Close: Suspend::platform_end(state); pm_suspend_target_state = Pm_Suspend_On.
14. return last err.
15. Recover_platform: Suspend::platform_recover(state); goto Resume_devices.

`Suspend::enter(state, *wakeup) -> Result<()>`:
1. Suspend::platform_prepare(state)?  /* unwind label: Platform_finish */
2. dpm_suspend_late(PMSG_SUSPEND).
3. on Err: pr_err("late suspend of devices failed"); goto Platform_finish.
4. Suspend::platform_prepare_late(state).
5. on Err: goto Devices_early_resume.
6. dpm_suspend_noirq(PMSG_SUSPEND).
7. on Err: pr_err("noirq suspend of devices failed"); goto Platform_early_resume.
8. Suspend::platform_prepare_noirq(state).
9. on Err: goto Platform_wake.
10. if suspend_test(TEST_PLATFORM): goto Platform_wake.
11. if state == Pm_Suspend_To_Idle: S2Idle::loop(); goto Platform_wake.
12. pm_sleep_disable_secondary_cpus().
13. on Err ∨ suspend_test(TEST_CPUS): goto Enable_cpus.
14. Arch::suspend_disable_irqs(); BUG_ON(!irqs_disabled()).
15. system_state = SYSTEM_SUSPEND.
16. let core_err = syscore_suspend().
17. if core_err == Ok:
    a. *wakeup = pm_wakeup_pending().
    b. if !(suspend_test(TEST_CORE) || *wakeup):
       i. trace_suspend_resume("machine_suspend", state, true).
       ii. err = (suspend_ops.enter)(state).
       iii. trace_suspend_resume("machine_suspend", state, false).
    c. else if *wakeup: err = Err(EBUSY).
    d. syscore_resume().
18. system_state = SYSTEM_RUNNING.
19. Arch::suspend_enable_irqs(); BUG_ON(irqs_disabled()).
20. Enable_cpus: pm_sleep_enable_secondary_cpus().
21. Platform_wake: Suspend::platform_resume_noirq(state); dpm_resume_noirq(PMSG_RESUME).
22. Platform_early_resume: Suspend::platform_resume_early(state).
23. Devices_early_resume: dpm_resume_early(PMSG_RESUME).
24. Platform_finish: Suspend::platform_resume_finish(state); return err.

`Suspend::finish()`:
1. suspend_thaw_processes() — kthreads + userspace thawed.
2. filesystems_thaw().
3. pm_notifier_call_chain(PM_POST_SUSPEND).
4. pm_restore_console().

`S2Idle::loop()`:
1. pm_pr_dbg("suspend-to-idle").
2. loop:
   - if s2idle_ops.wake.is_some() ∧ (s2idle_ops.wake)(): break.
   - else if pm_wakeup_pending(): break.
   - if s2idle_ops.check.is_some(): (s2idle_ops.check)().
   - S2Idle::enter().
3. pm_pr_dbg("resume from suspend-to-idle").

`S2Idle::enter()`:
1. trace_suspend_resume("machine_suspend", Pm_Suspend_To_Idle, true).
2. S2IDLE_LOCK.lock_irq().
3. if pm_wakeup_pending(): goto out.
4. s2idle_state = S2IDLE_STATE_ENTER.
5. S2IDLE_LOCK.unlock_irq().
6. wake_up_all_idle_cpus() — kick each CPU into idle.
7. swait_event_exclusive(&S2IDLE_WAIT_HEAD, s2idle_state == S2IDLE_STATE_WAKE).
8. wake_up_all_idle_cpus() — wake all CPUs to resume timers.
9. S2IDLE_LOCK.lock_irq().
10. out: s2idle_state = S2IDLE_STATE_NONE.
11. S2IDLE_LOCK.unlock_irq().
12. trace_suspend_resume("machine_suspend", Pm_Suspend_To_Idle, false).

`S2Idle::wake()` — wake-source IRQ context:
1. S2IDLE_LOCK.lock_irqsave().
2. if s2idle_state > S2IDLE_STATE_NONE: s2idle_state = S2IDLE_STATE_WAKE; swake_up_one(&S2IDLE_WAIT_HEAD).
3. S2IDLE_LOCK.unlock_irqrestore().

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `system_transition_mutex_serializes` | INVARIANT | per-enter_state: at most one pm_suspend in-flight system-wide; second attempt returns -EBUSY. |
| `irq_disabled_before_machine_enter` | INVARIANT | per-suspend_enter S1/S3: BUG_ON(!irqs_disabled) preceding suspend_ops->enter. |
| `irq_enabled_after_resume` | INVARIANT | per-suspend_enter: BUG_ON(irqs_disabled) after arch_suspend_enable_irqs. |
| `s2i_skips_arch_irq_disable` | INVARIANT | per-suspend_enter: state == PM_SUSPEND_TO_IDLE branch goes through s2idle_loop without calling arch_suspend_disable_irqs / syscore_suspend. |
| `wakeup_checked_at_every_phase` | INVARIANT | per-suspend pipeline: pm_wakeup_pending called at: pre-noirq-enter (s2idle_enter), post-syscore_suspend, s2idle_loop head. |
| `unwind_pairs_complete` | INVARIANT | per-suspend_enter: each forward step has matching unwind label (dpm_suspend_late ↔ dpm_resume_early, dpm_suspend_noirq ↔ dpm_resume_noirq, platform_prepare ↔ platform_resume_finish, secondary_cpus_disable ↔ _enable). |
| `s2idle_lock_held_across_state_xition` | INVARIANT | per-s2idle_enter: s2idle_state transitions to ENTER and to NONE under s2idle_lock. |
| `valid_state_required` | INVARIANT | per-enter_state: !S2I and !valid_state → -EINVAL before mutex. |

### Layer 2: TLA+

`kernel/power/suspend.tla`:
- States: ON → ENTER_STATE → PREPARE → DEVICES_AND_ENTER → SUSPEND_ENTER → (S2I_LOOP | MACHINE_ENTER) → RESUME → FINISH → ON.
- Properties:
  - `safety_irq_off_only_during_machine_enter` — per-suspend_enter: irqs disabled iff in (syscore_suspend ∪ suspend_ops.enter ∪ syscore_resume).
  - `safety_secondary_cpus_offline_only_during_machine_enter` — per-suspend_enter S1/S3: secondary CPUs offline exactly between pm_sleep_disable_secondary_cpus and pm_sleep_enable_secondary_cpus.
  - `safety_freeze_pairs_with_thaw` — per-pipeline: every successful suspend_freeze_processes pairs with suspend_thaw_processes; every failed path pairs with thaw + filesystems_thaw on the error rollback.
  - `safety_notifier_robust` — per-prepare: PM_SUSPEND_PREPARE_robust + PM_POST_SUSPEND on failure ensures notifier chain restored.
  - `safety_target_state_tracks_actual` — per-devices_and_enter: pm_suspend_target_state == state during the call, restored to PM_SUSPEND_ON at Close.
  - `liveness_wakeup_eventually_terminates_s2idle` — per-S2I: a wake event (s2idle_wake) eventually causes s2idle_loop to exit and the pipeline to resume.
  - `liveness_failed_step_unwinds` — per-any-error: control eventually reaches Resume_devices/Platform_finish/Finish/Unlock.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Suspend::pm_suspend` post: state ∈ [PM_SUSPEND_TO_IDLE..PM_SUSPEND_MAX); else Err(EINVAL). | `Suspend::pm_suspend` |
| `Suspend::enter_state` post: SYSTEM_TRANSITION released on every exit (Ok and Err). | `Suspend::enter_state` |
| `Suspend::prepare` post: on Err, console restored + notifier chain restored + filesystems thawed. | `Suspend::prepare` |
| `Suspend::devices_and_enter` post: on Err, dpm_resume_end called + console_resume_all called + platform_end called + pm_suspend_target_state = PM_SUSPEND_ON. | `Suspend::devices_and_enter` |
| `Suspend::enter` post: every successful dpm_suspend_{late,noirq} paired with corresponding dpm_resume_*; CPU hot-plug paired. | `Suspend::enter` |
| `S2Idle::enter` post: s2idle_state ∈ {NONE} on exit; s2idle_lock held during state transitions. | `S2Idle::enter` |
| `S2Idle::wake` post: only transitions ENTER→WAKE; no-op for NONE. | `S2Idle::wake` |
| `Suspend::valid_state` post: returns true ⟹ suspend_ops registered ∧ ops.valid(state) == nonzero ∧ ops.enter present. | `Suspend::valid_state` |

### Layer 4: Verus/Creusot functional

`Per-pm_suspend(state): enter_state → trylock → s2idle_begin (S2I) → fs_sync → clear_flags → prepare (notifiers + freeze) → devices_and_enter [platform_begin → console_suspend_all → dpm_suspend_start → loop(suspend_enter: platform_prepare → dpm_suspend_late → platform_prepare_late → dpm_suspend_noirq → platform_prepare_noirq → [S2I: s2idle_loop | S1/S3: disable_secondary_cpus → arch_disable_irqs → syscore_suspend → wakeup_pending? → suspend_ops.enter → syscore_resume → arch_enable_irqs → enable_secondary_cpus] → dpm_resume_noirq → dpm_resume_early → platform_resume_finish) until !suspend_again] → dpm_resume_end → console_resume_all → platform_end → finish (thaw + filesystems_thaw + PM_POST_SUSPEND + restore_console) → mutex_unlock` semantic equivalence — per-Documentation/admin-guide/pm/sleep-states.rst, Documentation/driver-api/pm/devices.rst.

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` and `kernel/power/00-overview.md` § Hardening.)

Suspend reinforcement:

- **Per-system_transition_mutex try-lock** — defense against per-concurrent-suspend races between sysfs writers / autosleep / ACPI.
- **Per-state-bounds check at pm_suspend** — defense against per-OOB suspend_state_t from sysfs or syscall.
- **Per-pm_wakeup_pending polled at every phase** — defense against per-wakeup-event-loss (wake arriving during prepare must abort cleanly).
- **Per-strict pairing of dpm_suspend_*/dpm_resume_***  — defense against per-half-suspended-device tree on error.
- **Per-BUG_ON(!irqs_disabled before machine_enter)** — defense against per-IRQ-during-low-power UAF.
- **Per-system_state = SYSTEM_SUSPEND fence** — defense against per-printk-flood / per-sched-tick during atomic suspend.
- **Per-syscore_suspend before machine_enter** — defense against per-clocksource-drift / per-timer-fire mid-low-power.
- **Per-S2I never disables arch IRQs** — defense against per-arch-IRQ-stall regression in S2I path.
- **Per-S2I lock-ordering (s2idle_lock before pending-check)** — defense against per-wakeup-event lost between pm_wakeup_pending and state assignment.
- **Per-platform_recover on dpm_suspend_start failure** — defense against per-platform left-in-half-suspended state.
- **Per-suspend_freeze_processes failure → filesystems_thaw + restore_console** — defense against per-frozen-fs deadlock on resume.
- **Per-cxl_mem_active() blocks S1/S3** — defense against per-CXL-memory-loss during deep sleep.
- **Per-PM_SUSPEND_PREPARE robust notifier (pairs with PM_POST_SUSPEND on failure)** — defense against per-notifier-leaks asymmetric state.
- **Per-mutex_unlock in every Unlock path** — defense against per-stuck-system_transition_mutex (no future suspend possible).

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy.
- **PAX_KERNEXEC** — W^X for any executable mapping.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization.
- **PAX_REFCOUNT** — saturating refcount on subsystem structs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for sensitive allocations.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on vtables.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding.
- **GRKERNSEC_DMESG** — syslog restriction.
- **CAP_SYS_ADMIN strict at /sys/power/state write** — enforced against the init user namespace; userns-root cannot trigger system-wide suspend.
- **dpm sequence integrity via PAX_RAP** — every `dev->bus/class/type->pm->{suspend,resume,...}` callback dispatched through kCFI; a tampered `dev_pm_ops` table fails the signature check before the device sees the call.
- **Hibernation key under PAX_MEMORY_SANITIZE** — the swap-encryption key buffer is zero-on-free, and the in-memory copy is wiped immediately after the snapshot is encrypted and written to swap.
- **system_transition_mutex owner audited** — `mutex_trylock(&system_transition_mutex)` failures logged via GRKERNSEC_DMESG-restricted log so a stuck-mutex DoS is observable to admins only.
- **suspend_ops indirect dispatch under PAX_RAP** — `suspend_ops->valid/begin/prepare/enter/wake/finish/end` all kCFI-signed.
- **/sys/power/disk and /sys/power/state PAX_USERCOPY** — sysfs writes bounded by the small mode-string length.
- **GRKERNSEC_HIDESYM on /sys/power/* address-leaking files** — any debug node that exposed kernel pointers (rare but historically present) sanitized for unprivileged readers.

Per-doc rationale: suspend/resume runs with much of the kernel quiesced, IRQs off, and CPUs hotplugged out — exactly the moment when a tampered indirect call is hardest to debug and easiest to weaponize for arbitrary kernel-mode execution. PAX_RAP/kCFI on the device-PM ops and the `suspend_ops` table is the load-bearing defense; CAP_SYS_ADMIN strictness shuts the userns-confined trigger; PAX_MEMORY_SANITIZE on the hibernation key prevents the most direct memory-disclosure attack against a swapped snapshot.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `kernel/power/main.c` sysfs `/sys/power/state` + `/sys/power/mem_sleep` parsing (covered separately in `kernel/power/main.md` Tier-3 if expanded).
- `kernel/power/process.c` freeze/thaw process internals (covered in `kernel/power/process.md` if expanded).
- `drivers/base/power/main.c` dpm_* device-PM walk (covered in `drivers/base/pm-system.md` Tier-3).
- `drivers/base/syscore.c` syscore_ops chain (covered in `drivers/base/syscore.md` if expanded).
- `kernel/power/hibernate.c` S4 / suspend-to-disk (covered in `kernel/power/hibernate.md` Tier-3 if expanded).
- `kernel/power/autosleep.c` autosleep workqueue (covered in `kernel/power/autosleep.md` if expanded).
- `kernel/power/wakeup_reason.c` wake-source reporting (covered in `kernel/power/wakeup-reason.md` if expanded).
- ACPI S-state encoding / _SS / _S3 / GPE wake routing (covered in `drivers/acpi/sleep.md` if expanded).
- arch `arch_suspend_disable_irqs` / FPU save / TSC freeze per-arch implementations (covered in respective arch Tier-3s).
- Implementation code.
