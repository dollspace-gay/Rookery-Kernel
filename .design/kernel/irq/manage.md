# Tier-3: kernel/irq/manage.c — IRQ runtime management

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/irq/00-overview.md
upstream-paths:
  - kernel/irq/manage.c (~2803 lines)
  - kernel/irq/pm.c (suspend_device_irqs, resume_device_irqs — exported)
  - kernel/irq/proc.c (/proc/irq/N — registration hooked here)
  - include/linux/interrupt.h (IRQF_*, IRQTF_*, irq_handler_t, request_irq, request_threaded_irq, free_irq, struct irqaction)
-->

## Summary

Runtime IRQ management is the userland-of-the-kernel surface that drivers actually call. Per-`request_threaded_irq(irq, handler, thread_fn, flags, name, dev_id)` allocates a `struct irqaction`, validates per-shared / per-trigger / per-oneshot semantics, kicks off a per-action kthread if `thread_fn != NULL` or force-threading is on, and links the action into `desc.action` chain. Per-`free_irq(irq, dev_id)` unlinks, stops the kthread, and shuts down the chip if last. Per-IRQF_SHARED enables ORed-handler shared lines (PCI-INTx, legacy ISA). Per-IRQF_ONESHOT keeps the line masked across handler+thread until thread completes (prevents reentry on level-triggered). Per-force-threading (`threadirqs` cmdline) makes every non-NO_THREAD action threaded for PREEMPT_RT lineage. Per-NMI registration goes through `request_nmi` and bypasses the action-chain because NMIs cannot share. Per-`/proc/irq/N` exposes affinity / smp_affinity / smp_affinity_list / spurious / effective_affinity / actions to userspace.

This Tier-3 covers `kernel/irq/manage.c` (~2803 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct irqaction` | per-handler descriptor | `IrqAction` |
| `request_threaded_irq()` | per-register | `Irq::request_threaded` |
| `request_irq()` (inline wrapper) | per-register short-hand | `Irq::request` |
| `request_any_context_irq()` | per-context-flexible request | `Irq::request_any_context` |
| `request_nmi()` | per-NMI register | `Irq::request_nmi` |
| `free_irq()` | per-deregister | `Irq::free` |
| `__free_irq()` | per-unlink | `Irq::free_inner` |
| `__cleanup_nmi()` | per-NMI deregister | `Irq::cleanup_nmi` |
| `__setup_irq()` | per-install | `Irq::setup` |
| `setup_irq()` (removed but invariants apply) | shared-irq install | merged into `Irq::setup` |
| `__irq_set_trigger()` | per-trigger program | `Irq::set_trigger` |
| `synchronize_irq()` / `synchronize_hardirq()` | per-quiesce | `Irq::synchronize` / `synchronize_hard` |
| `__synchronize_irq()` / `__synchronize_hardirq()` | per-quiesce-inner | `Irq::synchronize_inner` |
| `disable_irq()` / `disable_irq_nosync()` / `enable_irq()` | per-runtime gating | `Irq::disable` / `disable_nosync` / `enable` |
| `disable_hardirq()` | per-hardirq-disable (return-true-if-not-in-progress) | `Irq::disable_hardirq` |
| `disable_nmi_nosync()` / `enable_nmi()` | per-NMI gating | `Irq::disable_nmi_nosync` / `enable_nmi` |
| `__disable_irq()` / `__enable_irq()` | per-disable/enable inner | `Irq::disable_inner` / `enable_inner` |
| `__disable_irq_nosync()` | per-nosync disable | `Irq::disable_nosync_inner` |
| `irq_set_irq_wake()` | per-wakeup source | `Irq::set_irq_wake` |
| `set_irq_wake_real()` | per-chip wake program | `Irq::set_wake_chip` |
| `can_request_irq()` | per-availability check | `Irq::can_request` |
| `irq_set_affinity()` / `irq_force_affinity()` | per-affinity write | `Irq::set_affinity` / `force_affinity` |
| `irq_set_affinity_locked()` | per-affinity inner | `Irq::set_affinity_locked` |
| `irq_do_set_affinity()` | per-chip-program | `Irq::do_set_affinity` |
| `irq_try_set_affinity()` | per-try-affinity | `Irq::try_set_affinity` |
| `irq_set_affinity_pending()` | per-deferred affinity (in-progress) | `Irq::set_affinity_pending` |
| `irq_set_affinity_deactivated()` | per-affinity-while-shutdown | `Irq::set_affinity_deactivated` |
| `irq_set_affinity_notifier()` | per-notifier register | `Irq::set_affinity_notifier` |
| `irq_affinity_notify()` (workqueue) | per-deliver notify | `Irq::affinity_notify_work` |
| `irq_affinity_schedule_notify_work()` | per-schedule notify | `Irq::affinity_schedule_notify_work` |
| `__irq_apply_affinity_hint()` | per-hint apply | `Irq::apply_affinity_hint_inner` |
| `irq_can_set_affinity()` / `irq_can_set_affinity_usr()` | per-permit check | `Irq::can_set_affinity` / `can_set_affinity_usr` |
| `__irq_can_set_affinity()` | per-permit inner | `Irq::can_set_affinity_inner` |
| `irq_set_thread_affinity()` | per-thread affinity sync | `Irq::set_thread_affinity` |
| `irq_validate_effective_affinity()` | per-validate effective | `Irq::validate_effective_affinity` |
| `irq_setup_affinity()` | per-initial affinity | `Irq::setup_affinity` |
| `irq_update_affinity_desc()` | per-managed-affinity-desc | `Irq::update_affinity_desc` |
| `irq_set_vcpu_affinity()` | per-vCPU affinity | `Irq::set_vcpu_affinity` |
| `irq_set_parent()` | per-parent irq (chained) | `Irq::set_parent` |
| `irq_default_primary_handler()` | per-default primary stub | `Irq::default_primary_handler` |
| `irq_nested_primary_handler()` | per-nested primary stub | `Irq::nested_primary_handler` |
| `irq_forced_secondary_handler()` | per-forced-secondary stub | `Irq::forced_secondary_handler` |
| `irq_thread()` | per-action kthread | `Irq::thread_main` |
| `irq_thread_fn()` | per-handler invocation | `Irq::thread_fn` |
| `irq_forced_thread_fn()` | per-forced-threading fn | `Irq::forced_thread_fn` |
| `irq_thread_check_affinity()` | per-thread reaffinitize | `Irq::thread_check_affinity` |
| `irq_wait_for_interrupt()` | per-thread sleep | `Irq::wait_for_interrupt` |
| `irq_finalize_oneshot()` | per-oneshot unmask | `Irq::finalize_oneshot` |
| `irq_thread_dtor()` | per-kthread exit hook | `Irq::thread_dtor` |
| `irq_wake_secondary()` | per-wake secondary action | `Irq::wake_secondary` |
| `irq_thread_set_ready()` / `wake_up_and_wait_for_irq_thread_ready()` | per-thread readiness | `Irq::thread_set_ready` / `wait_thread_ready` |
| `wake_threads_waitq()` | per-completion broadcast | `Irq::wake_threads_waitq` |
| `irq_wake_thread()` | per-driver wake hook | `Irq::wake_thread` |
| `irq_setup_forced_threading()` | per-force-threading install | `Irq::setup_forced_threading` |
| `setup_forced_irqthreads()` (cmdline) | per-cmdline boot flag | `Irq::set_force_threading_cmdline` |
| `irq_request_resources()` / `irq_release_resources()` | per-driver acquire chip resources | `Irq::request_resources` / `release_resources` |
| `irq_supports_nmi()` | per-NMI capability check | `Irq::supports_nmi` |
| `irq_nmi_setup()` / `irq_nmi_teardown()` | per-NMI install | `Irq::nmi_setup` / `nmi_teardown` |
| `enable_percpu_irq()` / `disable_percpu_irq()` | per-CPU IRQ runtime gating | `Irq::enable_percpu` / `disable_percpu` |
| `enable_percpu_nmi()` / `disable_percpu_nmi()` | per-CPU NMI gating | `Irq::enable_percpu_nmi` / `disable_percpu_nmi` |
| `irq_percpu_is_enabled()` | per-CPU runtime probe | `Irq::percpu_is_enabled` |
| `request_percpu_irq_affinity()` | per-CPU IRQ register | `Irq::request_percpu_affinity` |
| `request_percpu_nmi()` | per-CPU NMI register | `Irq::request_percpu_nmi` |
| `free_percpu_irq()` / `free_percpu_nmi()` | per-CPU IRQ deregister | `Irq::free_percpu` / `free_percpu_nmi` |
| `__free_percpu_irq()` | per-CPU IRQ unlink | `Irq::free_percpu_inner` |
| `create_percpu_irqaction()` | per-CPU action alloc | `Irq::create_percpu_irqaction` |
| `prepare_percpu_nmi()` / `teardown_percpu_nmi()` | per-CPU NMI bring-up | `Irq::prepare_percpu_nmi` / `teardown` |
| `irq_get_irqchip_state()` / `irq_set_irqchip_state()` | per-state introspection | `Irq::get_irqchip_state` / `set` |
| `__irq_get_irqchip_state()` | per-state inner | `Irq::get_irqchip_state_inner` |
| `irq_has_action()` | per-existence check | `Irq::has_action` |
| `irq_check_status_bit()` | per-status probe | `Irq::check_status_bit` |
| `valid_percpu_irqaction()` | per-percpu install validator | `Irq::valid_percpu_irqaction` |

## Compatibility contract

REQ-1: struct irqaction:
- handler: per-primary IRQ handler (hardirq context).
- thread_fn: per-secondary threaded handler.
- name / dev_id / percpu_dev_id: per-identity.
- next: per-action chain (IRQF_SHARED).
- thread: per-action kthread (NULL if no thread).
- secondary: per-forced-threaded primary's secondary irqaction.
- thread_flags: IRQTF_RUNTHREAD / IRQTF_WARNED / IRQTF_AFFINITY / IRQTF_FORCED_THREAD / IRQTF_READY.
- flags: IRQF_*.
- thread_mask: per-bit identifying this action in desc.threads_oneshot.
- irq: per-action irq line.

REQ-2: IRQF_* flags:
- IRQF_TRIGGER_RISING / FALLING / HIGH / LOW / NONE — trigger type.
- IRQF_SHARED — multiple actions on line; handlers polled in turn.
- IRQF_PROBE_SHARED — probing tolerant of shared mismatch.
- IRQF_TIMER — kernel timer interrupt; no force-thread.
- IRQF_PERCPU — per-CPU interrupt; no force-thread.
- IRQF_NOBALANCING — exclude from balancing.
- IRQF_IRQPOLL — eligible for irqpoll.
- IRQF_ONESHOT — keep line masked across handler+thread until thread finishes.
- IRQF_NO_SUSPEND — keep enabled across suspend.
- IRQF_FORCE_RESUME — force enable on resume.
- IRQF_NO_THREAD — exempt from force-threading.
- IRQF_EARLY_RESUME — resume in NOIRQ phase.
- IRQF_COND_SUSPEND — share-with-NO_SUSPEND allowed.
- IRQF_NO_AUTOEN — request_irq leaves disabled.
- IRQF_NO_DEBUG — never spurious-poll.
- IRQF_COND_ONESHOT — adopt ONESHOT from prior shared action.

REQ-3: IRQTF_* (action.thread_flags):
- IRQTF_RUNTHREAD — primary requests thread to run.
- IRQTF_WARNED — once-warning emitted.
- IRQTF_AFFINITY — re-affinitize on next sched-in.
- IRQTF_FORCED_THREAD — this action is force-threaded.
- IRQTF_READY — thread initialized and waitable.

REQ-4: request_threaded_irq(irq, handler, thread_fn, flags, name, dev_id):
- if irq == IRQ_NOTCONNECTED: return -ENOTCONN.
- if (flags & IRQF_SHARED) ∧ !dev_id: return -EINVAL.
- if !handler:
  - if !thread_fn: return -EINVAL.
  - handler = irq_default_primary_handler.
- desc = irq_to_desc(irq); if !desc ∨ irq_settings_status & _IRQ_NOREQUEST: return -EINVAL.
- /* Per-NO_THREAD chip rule */
- if !can_request_irq(irq, flags): return -EINVAL.
- action = kzalloc(struct irqaction).
- action.handler = handler; thread_fn; flags; name; dev_id.
- ret = irq_chip_pm_get(&desc.irq_data).
- ret = __setup_irq(irq, desc, action).
- if ret: irq_chip_pm_put; kfree(action); return ret.
- return 0.

REQ-5: __setup_irq(irq, desc, new):
- nested = irq_settings_is_nested_thread(desc).
- /* Per-NESTED */
- if nested:
  - if !new.thread_fn: return -EINVAL.
  - new.handler = irq_nested_primary_handler.
- elif irq_settings_can_thread(desc):
  - irq_setup_forced_threading(new).
- /* Per-thread create */
- if new.thread_fn ∧ !nested:
  - ret = setup_irq_thread(new, irq, false).
- if new.secondary:
  - ret = setup_irq_thread(new.secondary, irq, true).
- /* Per-NMI not allowed via request_irq path */
- if WARN_ON(new.flags & IRQF_NO_AUTOEN ∧ new.flags & IRQF_SHARED): ...
- raw_spin_lock_irqsave(&desc.lock).
- /* Per-shared / per-thread_mask negotiation */
- old = &desc.action.
- if *old:
  - /* Check IRQF_SHARED compatibility */
  - if !((*old).flags & IRQF_SHARED) ∨ !(new.flags & IRQF_SHARED):
    - if !(new.flags & IRQF_PROBE_SHARED): ret = -EBUSY; goto mismatch.
  - /* Trigger compatibility */
  - oldtype = (*old).flags & IRQF_TRIGGER_MASK.
  - if oldtype ∧ (oldtype != (new.flags & IRQF_TRIGGER_MASK)): ret = -EBUSY.
  - /* ONESHOT-cond / PERCPU compatibility */
  - ...
  - /* Compute thread_mask */
  - while (*old): old = &(*old).next; ...
- elif !shared:
  - /* First action — program trigger / activate / startup */
  - if new.flags & IRQF_TRIGGER_MASK:
    - __irq_set_trigger(desc, new.flags & IRQF_TRIGGER_MASK).
  - if !(new.flags & IRQF_NO_AUTOEN):
    - irq_activate_and_startup(desc, IRQ_RESEND).
- /* Insert */
- *old = new.
- raw_spin_unlock_irqrestore.
- if new.thread: wake_up_process(new.thread).
- if new.secondary: wake_up_process(new.secondary.thread).
- register_irq_proc(irq, desc).
- register_handler_proc(irq, new).

REQ-6: irq_setup_forced_threading(new):
- if !force_irqthreads(): return 0.
- if new.flags & (IRQF_NO_THREAD | IRQF_PERCPU | IRQF_ONESHOT): return 0.
- /* PRIMARY → forced-thread */
- new.flags |= IRQF_ONESHOT.
- new.thread_flags |= IRQTF_FORCED_THREAD.
- if new.handler != irq_default_primary_handler ∧ new.thread_fn:
  - new.secondary = kzalloc(struct irqaction).
  - new.secondary.handler = irq_forced_secondary_handler.
  - new.secondary.thread_fn = new.thread_fn.
  - new.secondary.dev_id = new.dev_id.
  - new.secondary.name = forced-name.
  - new.thread_fn = new.handler.
  - new.handler = irq_default_primary_handler.

REQ-7: irq_thread(data):
- action = data.
- /* Per-thread main loop */
- task_work_add(current, &on_exit_work) /* irq_thread_dtor */.
- irq_thread_set_ready(desc, action).
- loop:
  - if irq_wait_for_interrupt(desc, action): break.
  - irq_thread_check_affinity(desc, action).
  - if action.thread_flags & IRQTF_FORCED_THREAD:
    - action_ret = irq_forced_thread_fn(desc, action).
  - else:
    - action_ret = irq_thread_fn(desc, action).
  - if test_and_clear_bit(IRQTF_RUNTHREAD, &action.thread_flags):
    - wake_threads_waitq(desc).
- /* Per-exit: drop refs */
- return 0.

REQ-8: irq_thread_fn(desc, action):
- ret = action.thread_fn(action.irq, action.dev_id).
- if ret == IRQ_HANDLED: atomic_inc(&desc.threads_handled).
- irq_finalize_oneshot(desc, action).
- return ret.

REQ-9: irq_forced_thread_fn(desc, action):
- local_bh_disable.
- ret = action.thread_fn(action.irq, action.dev_id).
- if ret == IRQ_HANDLED: atomic_inc(&desc.threads_handled).
- irq_finalize_oneshot(desc, action).
- local_bh_enable.
- return ret.

REQ-10: irq_wait_for_interrupt(desc, action):
- loop:
  - set_current_state(TASK_INTERRUPTIBLE).
  - if kthread_should_stop():
    - if test_and_clear_bit(IRQTF_RUNTHREAD, ...): set_current_state(TASK_RUNNING); /* one final run */ return -1.
    - set_current_state(TASK_RUNNING); return -1.
  - if test_and_clear_bit(IRQTF_RUNTHREAD, &action.thread_flags):
    - set_current_state(TASK_RUNNING); return 0.
  - schedule.

REQ-11: irq_finalize_oneshot(desc, action):
- if !(desc.istate & IRQS_ONESHOT) ∨ !(action.thread_mask): return.
- chip_busy_wait. /* until chip ready */
- raw_spin_lock_irq(&desc.lock).
- /* Clear this action's thread_mask bit */
- desc.threads_oneshot &= ~action.thread_mask.
- /* If all done and not disabled, unmask */
- if desc.threads_oneshot == 0 ∧ !irqd_irq_disabled(&desc.irq_data) ∧ irqd_irq_masked(&desc.irq_data):
  - unmask_threaded_irq(desc).
- raw_spin_unlock_irq.

REQ-12: irq_wake_thread(irq, dev_id):
- desc = irq_to_desc(irq); if !desc: return.
- raw_spin_lock_irqsave(&desc.lock).
- for action in desc.action chain:
  - if action.dev_id == dev_id:
    - __irq_wake_thread(desc, action).
    - break.
- raw_spin_unlock_irqrestore.

REQ-13: __irq_wake_thread(desc, action):
- /* Called from handle_irq_event when handler returns IRQ_WAKE_THREAD */
- if test_and_set_bit(IRQTF_RUNTHREAD, &action.thread_flags): return.
- atomic_inc(&desc.threads_active).
- wake_up_process(action.thread).

REQ-14: free_irq(irq, dev_id) → __free_irq(desc, dev_id):
- mutex_lock(&desc.request_mutex).
- chip_bus_lock.
- raw_spin_lock_irqsave(&desc.lock).
- /* Find and unlink */
- action_ptr = &desc.action.
- for: action = *action_ptr; if !action: WARN_ON; ret = NULL; goto out.
- if action.dev_id == dev_id: break.
- action_ptr = &action.next.
- *action_ptr = action.next.
- /* Last action? */
- if !desc.action:
  - irq_settings_clr_disable_unlazy(desc).
  - irq_shutdown_and_deactivate(desc).
- raw_spin_unlock_irqrestore.
- unregister_handler_proc(irq, action).
- /* Wait for in-progress handler + thread to drain */
- __synchronize_irq(desc).
- if action.thread: kthread_stop(action.thread); put_task_struct(action.thread).
- if action.secondary ∧ action.secondary.thread: kthread_stop(action.secondary.thread); put_task_struct(action.secondary.thread); kfree(action.secondary).
- irq_chip_pm_put(&desc.irq_data).
- mutex_unlock(&desc.request_mutex).
- chip_bus_sync_unlock.
- return action.

REQ-15: synchronize_irq(irq) / synchronize_hardirq(irq):
- desc = irq_to_desc(irq); if !desc: return.
- /* hardirq: wait for IRQD_IRQ_INPROGRESS */
- /* irq: wait for hardirq + threads_active == 0 */
- __synchronize_hardirq(desc, sync_chip=true).
- __synchronize_irq(desc): wait_event(desc.wait_for_threads, atomic_read(&desc.threads_active) == 0).

REQ-16: __synchronize_hardirq(desc, sync_chip):
- /* Per-loop: wait until !IRQD_IRQ_INPROGRESS and chip not pending if sync_chip */
- do:
  - inprogress = irq_wait_on_inprogress(desc).
- while inprogress.

REQ-17: disable_irq(irq) / disable_irq_nosync(irq) / enable_irq(irq):
- disable_irq_nosync: __disable_irq_nosync(irq).
- disable_irq: disable_irq_nosync(irq); synchronize_irq(irq).
- enable_irq: __enable_irq(desc); irq_chip_pm_put.

REQ-18: disable_hardirq(irq):
- /* Returns false if !sync; caller wraps. */
- disable_irq_nosync(irq).
- return synchronize_hardirq(irq).

REQ-19: __enable_irq(desc):
- if !desc.depth: WARN("Unbalanced enable_irq"); return.
- desc.depth--.
- if desc.depth == 0:
  - if irq_settings_is_disabled_unlazy(desc): __irq_startup(desc).
  - else: irq_enable(desc).
  - check_irq_resend(desc, false).

REQ-20: irq_set_irq_wake(irq, on):
- desc = irq_get_desc_buslock(irq, &flags, IRQ_GET_DESC_CHECK_GLOBAL).
- if on:
  - if desc.wake_depth++ == 0:
    - ret = set_irq_wake_real(irq, on).
    - if ret: desc.wake_depth--; ...
    - else: irqd_set(d, IRQD_WAKEUP_STATE).
- else:
  - if --desc.wake_depth == 0:
    - set_irq_wake_real(irq, on).
    - irqd_clear(d, IRQD_WAKEUP_STATE).
- irq_put_desc_busunlock.

REQ-21: irq_set_affinity(irq, mask) / irq_force_affinity(irq, mask):
- __irq_set_affinity(irq, mask, force).

REQ-22: __irq_set_affinity(irq, mask, force):
- desc = irq_get_desc_buslock(irq, &flags, IRQ_GET_DESC_CHECK_GLOBAL).
- if !irq_can_set_affinity(irq): return -EIO.
- ret = irq_set_affinity_locked(&desc.irq_data, mask, force).
- irq_put_desc_busunlock.
- return ret.

REQ-23: irq_set_affinity_locked(data, mask, force):
- if irq_set_affinity_deactivated(data, mask): return 0.
- if irq_move_pending(data) ∧ !force:
  - irq_set_affinity_pending(data, mask).
  - return 0.
- ret = irq_try_set_affinity(data, mask, force).
- /* Notify */
- if ret >= 0:
  - cpumask_copy(desc.irq_common_data.affinity, mask).
  - irq_set_thread_affinity(desc).
  - irq_affinity_schedule_notify_work(desc).
- return ret.

REQ-24: irq_do_set_affinity(data, mask, force):
- chip = irq_data_get_irq_chip(data).
- if !chip ∨ !chip.irq_set_affinity: return -EINVAL.
- ret = chip.irq_set_affinity(data, mask, force).
- switch ret:
  - IRQ_SET_MASK_OK / IRQ_SET_MASK_OK_DONE: cpumask_copy.
  - IRQ_SET_MASK_OK_NOCOPY: ret = 0.

REQ-25: irq_setup_affinity(desc):
- if !__irq_can_set_affinity(desc): return 0.
- /* Compute new mask from possible × node × default */
- cpumask_and(&mask, desc.irq_common_data.affinity, cpu_online_mask).
- if hk_flags: cpumask_and(&mask, &mask, housekeeping_cpumask(...)).
- if cpumask_empty(&mask): cpumask_and(&mask, ..., cpu_online_mask).
- return irq_do_set_affinity(d, &mask, false).

REQ-26: __irq_apply_affinity_hint(irq, m, setaffinity):
- desc.affinity_hint = m.
- if setaffinity: __irq_set_affinity(irq, m, false).

REQ-27: irq_set_affinity_notifier(irq, notify):
- desc = irq_to_desc(irq).
- old = desc.affinity_notify; if old: kref_put(&old.kref, old.release).
- if notify: kref_init(&notify.kref); INIT_WORK(&notify.work, irq_affinity_notify).
- desc.affinity_notify = notify.

REQ-28: request_nmi(irq, handler, flags, name, dev_id):
- /* No shared NMIs, no threaded NMIs */
- if !(flags & IRQF_PERCPU) ∧ irq is not NMI-capable: return -EINVAL.
- if flags & (IRQF_SHARED | IRQF_COND_SUSPEND) ∨ !handler: return -EINVAL.
- action = kzalloc(irqaction).
- desc.istate |= IRQS_NMI.
- ret = irq_nmi_setup(desc) /* chip.irq_nmi_setup */.
- __setup_irq(irq, desc, action) /* fast-path; no thread */.
- enable line; chip's NMI dispatch path used in handle_fasteoi_nmi.

REQ-29: irq_nmi_setup(desc) / irq_nmi_teardown(desc):
- chip = desc.irq_data.chip.
- if !chip.irq_nmi_setup: return -EINVAL.
- ret = chip.irq_nmi_setup(&desc.irq_data); if ret: return ret.
- irq_state_clr_masked; irq_state_clr_disabled.
- /* Teardown reverses + chip.irq_nmi_teardown. */

REQ-30: request_percpu_irq_affinity / request_percpu_nmi / free_percpu_irq / free_percpu_nmi:
- Per-CPU IRQs share a single irqaction with percpu_dev_id pointing at __percpu region.
- create_percpu_irqaction allocates and binds.
- enable_percpu_irq programs trigger and unmasks on this CPU.

REQ-31: enable_percpu_irq(irq, type) / disable_percpu_irq(irq):
- desc = irq_get_desc_buslock(irq, &flags, IRQ_GET_DESC_CHECK_PERCPU).
- if type ∧ type != irqd_get_trigger_type(&desc.irq_data): __irq_set_trigger(desc, type).
- irq_percpu_enable(desc, smp_processor_id()).
- desc.action: action.percpu_enabled.set(this_cpu).
- /* disable: irq_percpu_disable + clear-per-cpu-flag */

REQ-32: irq_get_irqchip_state(irq, which, &state):
- desc = irq_get_desc_buslock(irq, &flags, IRQ_GET_DESC_CHECK_GLOBAL).
- ret = __irq_get_irqchip_state(&desc.irq_data, which, &state).
- /* Walks via chip.irq_get_irqchip_state or up the hierarchy */

REQ-33: irq_set_irqchip_state(irq, which, val):
- analogous; chip.irq_set_irqchip_state(d, which, val).

REQ-34: /proc/irq/N (kernel/irq/proc.c — registered from manage.c via register_irq_proc / register_handler_proc):
- /proc/irq/N/smp_affinity, smp_affinity_list — write-via irq_can_set_affinity_usr → irq_set_affinity.
- /proc/irq/N/effective_affinity, effective_affinity_list — read-only chip-effective mask.
- /proc/irq/N/affinity_hint — read-only desc.affinity_hint.
- /proc/irq/N/spurious — read-only spurious counters.
- /proc/irq/N/<name> — per-action subdirs.
- /proc/irq/default_smp_affinity — global default.

REQ-35: suspend/resume IRQ (delegated to kernel/irq/pm.c):
- suspend_device_irqs: for each desc: irq_suspend_one(desc) — if IRQF_NO_SUSPEND or wake-source, leave armed; else __disable_irq.
- resume_device_irqs: re-enable.
- IRQF_EARLY_RESUME taken in noirq-resume.
- IRQF_COND_SUSPEND lets one action's IRQF_NO_SUSPEND not poison its shared peers.

REQ-36: force-threading:
- setup_forced_irqthreads("threadirqs"): force_irqthreads_key.enable().
- request flow: irq_setup_forced_threading promotes primary to threaded ONESHOT + secondary irqaction.
- IRQF_NO_THREAD, IRQF_PERCPU, IRQF_ONESHOT exempt.

REQ-37: irq_thread_set_ready / wake_up_and_wait_for_irq_thread_ready:
- /* Sync request_irq with thread bring-up */
- set_bit(IRQTF_READY, &action.thread_flags); wake_up(&desc.wait_for_threads).
- wake_up_and_wait_for_irq_thread_ready: wake_up_process(action.thread); wait_event(desc.wait_for_threads, IRQTF_READY).

REQ-38: irq_thread_dtor(unused):
- task_work callback at thread-exit: decrement desc.threads_active if IRQTF_RUNTHREAD set; wake waiters.

REQ-39: bad_chained_irq / chained_action sentinel — see chip.md REQ-34.

REQ-40: can_request_irq(irq, irqflags):
- desc = irq_to_desc(irq).
- if !desc ∨ irq_settings_can_request(desc) is false: return false.
- if !desc.action: return true.
- if !((desc.action.flags & irqflags) & IRQF_SHARED): return false.
- if (desc.action.flags & IRQF_TRIGGER_MASK) != (irqflags & IRQF_TRIGGER_MASK): return false.
- return true.

REQ-41: irq_update_affinity_desc(irq, affinity):
- /* Managed-IRQ affinity update (e.g. MSI-X) */
- desc = irq_get_desc_buslock(irq, &flags, IRQ_GET_DESC_CHECK_GLOBAL).
- if !affinity ∨ cpumask_empty(&affinity.mask): irqd_clr_managed_shutdown(d).
- else: cpumask_copy(desc.irq_common_data.affinity, &affinity.mask); irqd_set_managed_shutdown.

REQ-42: irq_validate_effective_affinity:
- /* CONFIG_GENERIC_IRQ_EFFECTIVE_AFF_MASK only */
- WARN_ON_ONCE(cpumask_empty(d.common.effective_affinity)).

REQ-43: irq_set_vcpu_affinity(irq, vcpu_info):
- chip.irq_set_vcpu_affinity(d, vcpu_info) — KVM-pass-through (Intel-PI, GICv4-VLPI).

REQ-44: request_any_context_irq:
- request_irq with thread_fn=NULL first; if chip is nested-thread, retry as threaded.
- Returns IRQC_IS_HARDIRQ or IRQC_IS_NESTED.

## Acceptance Criteria

- [ ] AC-1: request_threaded_irq with handler=NULL ∧ thread_fn=NULL: -EINVAL.
- [ ] AC-2: request_threaded_irq with IRQF_SHARED ∧ dev_id=NULL: -EINVAL.
- [ ] AC-3: Two request_irq's: second must match IRQF_SHARED ∧ trigger ∨ -EBUSY.
- [ ] AC-4: IRQF_ONESHOT: line stays masked until thread finishes (irq_finalize_oneshot).
- [ ] AC-5: free_irq waits for in-progress handler and thread (synchronize_irq inside).
- [ ] AC-6: free_irq on last action: chip shutdown_and_deactivate runs.
- [ ] AC-7: force-threading promotes primary to secondary + ONESHOT.
- [ ] AC-8: IRQF_NO_THREAD / IRQF_PERCPU / IRQF_ONESHOT exempt from force-threading.
- [ ] AC-9: disable_irq is balanced by enable_irq (desc.depth).
- [ ] AC-10: disable_irq_nosync does not call synchronize_irq.
- [ ] AC-11: synchronize_hardirq waits only for IRQD_IRQ_INPROGRESS; not threads.
- [ ] AC-12: synchronize_irq waits for hardirq + threads_active==0.
- [ ] AC-13: irq_set_affinity rejects irq with !chip.irq_set_affinity: -EINVAL.
- [ ] AC-14: irq_set_affinity_notifier: previous notifier's kref dropped.
- [ ] AC-15: request_nmi rejects IRQF_SHARED, threaded handler, sleeping-allowed flags.
- [ ] AC-16: enable_percpu_irq only acts on the current CPU.
- [ ] AC-17: free_percpu_irq fails if any CPU still has the irq enabled.
- [ ] AC-18: irq_set_irq_wake: nested wake_depth refcount; chip.irq_set_wake called only at 0→1 / 1→0.
- [ ] AC-19: /proc/irq/N/smp_affinity write: validates irq_can_set_affinity_usr; -EIO if not.
- [ ] AC-20: suspend_device_irqs: IRQF_NO_SUSPEND lines remain armed.
- [ ] AC-21: resume_device_irqs: IRQF_EARLY_RESUME in noirq phase.
- [ ] AC-22: irq_wake_thread sets IRQTF_RUNTHREAD and wakes action.thread.
- [ ] AC-23: irq_thread terminates only on kthread_should_stop; runs one final iter if pending.

## Architecture

```
struct IrqAction {
  handler: irq_handler_t,
  thread_fn: Option<irq_handler_t>,
  name: &'static str,
  dev_id: *mut (),
  percpu_dev_id: *mut (),         // for percpu
  next: Option<*mut IrqAction>,
  thread: Option<TaskRef>,        // kthread
  secondary: Option<Box<IrqAction>>,
  thread_flags: AtomicU64,         // IRQTF_*
  thread_mask: u64,
  flags: u32,                      // IRQF_*
  irq: u32,
}
```

`Irq::request_threaded(irq, handler, thread_fn, flags, name, dev_id) -> Result<(), Errno>`:
1. if irq == IRQ_NOTCONNECTED: return Err(ENOTCONN).
2. if (flags & IRQF_SHARED) ∧ dev_id.is_null(): return Err(EINVAL).
3. if handler.is_none():
   - if thread_fn.is_none(): return Err(EINVAL).
   - handler = Some(irq_default_primary_handler).
4. desc = irq_to_desc(irq).ok_or(EINVAL)?.
5. if irq_settings_can_request(desc) == false: return Err(EINVAL).
6. if !can_request_irq(irq, flags): return Err(EINVAL).
7. action = Box::new(IrqAction { handler, thread_fn, flags, name, dev_id, ... }).
8. Irq::chip_pm_get(&desc.irq_data)?.
9. Irq::setup(irq, desc, action).map_err(|e| { Irq::chip_pm_put(...); e })?.
10. Ok(()).

`Irq::setup(irq, desc, new)`:
1. nested = irq_settings_is_nested_thread(desc).
2. if nested:
   - if new.thread_fn.is_none(): return Err(EINVAL).
   - new.handler = irq_nested_primary_handler.
3. elif irq_settings_can_thread(desc):
   - irq_setup_forced_threading(new).
4. if new.thread_fn.is_some() ∧ !nested:
   - setup_irq_thread(new, irq, false)?.
5. if let Some(sec) = &mut new.secondary:
   - setup_irq_thread(sec, irq, true)?.
6. mutex_lock(&desc.request_mutex).
7. chip_bus_lock(desc).
8. raw_spin_lock_irqsave(&desc.lock).
9. /* Walk desc.action chain; validate SHARED/TRIGGER/ONESHOT/PERCPU compatibility */
10. /* Allocate thread_mask bit */
11. if desc.action.is_none():
    - if new.flags & IRQF_TRIGGER_MASK != 0: __irq_set_trigger(desc, new.flags & IRQF_TRIGGER_MASK)?.
    - if (new.flags & IRQF_NO_AUTOEN) == 0: irq_activate_and_startup(desc, IRQ_RESEND).
12. *tail = new.
13. raw_spin_unlock_irqrestore.
14. if let Some(t) = new.thread: wake_up_and_wait_for_irq_thread_ready(desc, new).
15. if let Some(s) = &new.secondary: wake_up_and_wait_for_irq_thread_ready(desc, s).
16. chip_bus_sync_unlock(desc).
17. mutex_unlock(&desc.request_mutex).
18. register_irq_proc(irq, desc).
19. register_handler_proc(irq, new).
20. Ok(()).

`Irq::thread_main(action)`:
1. task_work_add(current, &on_exit_work_irq_thread_dtor).
2. irq_thread_set_ready(desc, action).
3. loop {
4.   if irq_wait_for_interrupt(desc, action) < 0: break.
5.   irq_thread_check_affinity(desc, action).
6.   if action.thread_flags & IRQTF_FORCED_THREAD:
7.     ret = irq_forced_thread_fn(desc, action).
8.   else:
9.     ret = irq_thread_fn(desc, action).
10.  if test_and_clear_bit(IRQTF_RUNTHREAD, &action.thread_flags):
11.    wake_threads_waitq(desc).
12. }
13. /* on exit: irq_thread_dtor cleans active count */.

`Irq::wait_for_interrupt(desc, action)`:
1. loop {
2.   set_current_state(TASK_INTERRUPTIBLE).
3.   if kthread_should_stop():
4.     if test_and_clear_bit(IRQTF_RUNTHREAD, &action.thread_flags):
5.       set_current_state(TASK_RUNNING); return Ok(()) /* run one final */.
6.     set_current_state(TASK_RUNNING); return Err(()).
7.   if test_and_clear_bit(IRQTF_RUNTHREAD, &action.thread_flags):
8.     set_current_state(TASK_RUNNING); return Ok(()).
9.   schedule().
10. }.

`Irq::finalize_oneshot(desc, action)`:
1. if !(desc.istate & IRQS_ONESHOT) ∨ action.thread_mask == 0: return.
2. chip_bus_lock(desc) /* may sleep */.
3. raw_spin_lock_irq(&desc.lock).
4. desc.threads_oneshot &= !action.thread_mask.
5. if desc.threads_oneshot == 0 ∧ !irqd_irq_disabled(&desc.irq_data) ∧ irqd_irq_masked(&desc.irq_data):
   - unmask_threaded_irq(desc).
6. raw_spin_unlock_irq.
7. chip_bus_sync_unlock(desc).

`Irq::wake_thread(irq, dev_id)`:
1. desc = irq_to_desc(irq).ok_or(())?.
2. raw_spin_lock_irqsave(&desc.lock).
3. for action in desc.action.iter():
   - if action.dev_id == dev_id:
     - __irq_wake_thread(desc, action).
     - break.
4. raw_spin_unlock_irqrestore.

`Irq::free(irq, dev_id)`:
1. action = Irq::free_inner(desc, dev_id).
2. /* synchronize_irq is inside free_inner */.
3. if let Some(t) = action.thread: kthread_stop(t); put_task_struct(t).
4. if let Some(s) = action.secondary { if let Some(t) = s.thread { kthread_stop(t); put_task_struct(t); }; drop(s); }.
5. Irq::chip_pm_put(&desc.irq_data).
6. drop(action).

`Irq::free_inner(desc, dev_id) -> IrqAction`:
1. mutex_lock(&desc.request_mutex).
2. chip_bus_lock.
3. raw_spin_lock_irqsave(&desc.lock).
4. /* Walk chain; unlink */
5. /* If empty: irq_shutdown_and_deactivate(desc) */
6. raw_spin_unlock_irqrestore.
7. unregister_handler_proc(irq, action).
8. __synchronize_irq(desc).
9. chip_bus_sync_unlock.
10. mutex_unlock(&desc.request_mutex).
11. return action.

`Irq::synchronize(irq)`:
1. desc = irq_to_desc(irq).ok_or(())?.
2. __synchronize_hardirq(desc, true).
3. __synchronize_irq(desc): wait_event(desc.wait_for_threads, atomic_read(&desc.threads_active) == 0).

`Irq::set_affinity_locked(data, mask, force)`:
1. if irq_set_affinity_deactivated(data, mask): return Ok(()).
2. if irq_move_pending(data) ∧ !force:
   - irq_set_affinity_pending(data, mask).
   - return Ok(()).
3. ret = irq_try_set_affinity(data, mask, force).
4. if ret.is_ok():
   - cpumask_copy(desc.irq_common_data.affinity, mask).
   - irq_set_thread_affinity(desc).
   - irq_affinity_schedule_notify_work(desc).
5. ret.

`Irq::request_nmi(irq, handler, flags, name, dev_id)`:
1. /* Disallow IRQF_SHARED / IRQF_PERCPU(only NMI-cpu-id var) / no thread_fn */.
2. if flags & (IRQF_SHARED|IRQF_COND_SUSPEND) != 0: return Err(EINVAL).
3. if handler.is_none(): return Err(EINVAL).
4. if !Irq::supports_nmi(desc): return Err(EINVAL).
5. action = Box::new(IrqAction { handler, flags: flags|IRQF_NO_THREAD|IRQF_NOBALANCING, ... }).
6. desc.istate |= IRQS_NMI.
7. Irq::nmi_setup(desc)?.
8. Irq::setup_minimal(irq, desc, action)?.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `request_irq_paired_with_free_irq` | INVARIANT | per-action: every successful request has matching free. |
| `irq_thread_terminates_on_kthread_stop` | INVARIANT | per-irq_thread: kthread_should_stop ⟹ loop exits. |
| `oneshot_unmask_iff_all_threads_done` | INVARIANT | per-irq_finalize_oneshot: unmask only when desc.threads_oneshot == 0 ∧ !disabled. |
| `shared_chain_homogeneous_trigger` | INVARIANT | per-__setup_irq: all shared actions same IRQF_TRIGGER_MASK. |
| `shared_chain_all_or_none_shared` | INVARIANT | per-__setup_irq: no mixed IRQF_SHARED / non-shared. |
| `disable_enable_balanced` | INVARIANT | per-desc.depth: monotonic ≥ 0; reaches 0 ⟺ chip enabled. |
| `wake_depth_balanced` | INVARIANT | per-desc.wake_depth: monotonic ≥ 0; chip.irq_set_wake invoked only on 0↔1. |
| `nmi_no_thread` | INVARIANT | per-request_nmi: action.thread_fn == NULL ∧ action.thread == NULL. |
| `nmi_no_shared` | INVARIANT | per-request_nmi: IRQF_SHARED not set. |
| `threads_active_balanced` | INVARIANT | per-__irq_wake_thread inc ↔ per-irq_thread_fn / irq_finalize_oneshot dec ↔ per-irq_thread_dtor dec. |
| `forced_threading_oneshot` | INVARIANT | per-irq_setup_forced_threading: IRQF_ONESHOT set. |
| `forced_threading_secondary_iff_thread_fn` | INVARIANT | per-irq_setup_forced_threading: secondary allocated iff orig has thread_fn. |
| `synchronize_irq_zeroes_threads_active` | INVARIANT | per-synchronize_irq: returns only when desc.threads_active == 0. |
| `affinity_notifier_kref_balanced` | INVARIANT | per-irq_set_affinity_notifier: prior notifier's kref_put exactly once. |
| `percpu_enable_only_this_cpu` | INVARIANT | per-enable_percpu_irq: smp_processor_id() bit set, not others. |
| `setup_irq_under_desc_request_mutex` | INVARIANT | per-__setup_irq: desc.request_mutex held. |
| `proc_smp_affinity_write_validated` | INVARIANT | per-/proc/irq/N/smp_affinity: validates irq_can_set_affinity_usr first. |

### Layer 2: TLA+

`kernel/irq/manage.tla`:
- Per-action lifecycle: ALLOC → SETUP → ARMED → IRQ_RUN ↻ → FREE_PENDING → FREED.
- Per-thread lifecycle: SPAWNED → READY → IDLE → RUN ↻ → STOPPING → DEAD.
- Per-disable/enable depth counter.
- Per-wake_depth counter.
- Properties:
  - `safety_no_handler_after_free` — per-free: no further dispatch to this action.
  - `safety_oneshot_no_double_handler` — per-IRQF_ONESHOT: line masked from primary-return until thread-finish.
  - `safety_shared_all_match_trigger` — per-shared chain: trigger_type unanimous.
  - `safety_disable_depth_nonneg` — per-desc.depth: never < 0.
  - `safety_wake_depth_nonneg` — per-desc.wake_depth: never < 0.
  - `safety_synchronize_irq_drains` — per-synchronize_irq termination: all in-flight handlers + threads finished.
  - `liveness_request_threaded_irq_returns` — per-request: terminates.
  - `liveness_free_irq_returns` — per-free: terminates after synchronize_irq.
  - `liveness_wake_thread_runs_action` — per-IRQ_WAKE_THREAD: action.thread eventually runs thread_fn.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Irq::request_threaded` post: Ok ⟹ action linked in desc.action chain ∧ thread spawned iff thread_fn | `Irq::request_threaded` |
| `Irq::setup` post: action.thread_mask is unique bit in desc.threads_oneshot universe | `Irq::setup` |
| `Irq::free_inner` post: action.dev_id removed; chip shutdown iff empty | `Irq::free_inner` |
| `Irq::thread_main` post: runs at most one final iteration after kthread_should_stop | `Irq::thread_main` |
| `Irq::finalize_oneshot` post: unmask iff desc.threads_oneshot==0 ∧ !disabled ∧ masked | `Irq::finalize_oneshot` |
| `Irq::wake_thread` post: IRQTF_RUNTHREAD set on matching action | `Irq::wake_thread` |
| `Irq::synchronize` post: desc.threads_active == 0 ∧ !IRQD_IRQ_INPROGRESS | `Irq::synchronize` |
| `Irq::disable_inner` post: desc.depth incremented; chip masked when 0→1 | `Irq::disable_inner` |
| `Irq::enable_inner` post: desc.depth decremented; chip unmasked when 1→0 | `Irq::enable_inner` |
| `Irq::set_irq_wake` post: wake_depth touched exactly once; chip.irq_set_wake only on 0↔1 | `Irq::set_irq_wake` |
| `Irq::set_affinity_locked` post: notifier scheduled iff success | `Irq::set_affinity_locked` |
| `Irq::request_nmi` post: action.thread_fn==NULL ∧ IRQS_NMI set | `Irq::request_nmi` |
| `Irq::setup_forced_threading` post: IRQF_ONESHOT ∧ secondary alloc iff orig thread_fn | `Irq::setup_forced_threading` |

### Layer 4: Verus/Creusot functional

Per-request_threaded_irq → __setup_irq → irq_thread → handle_irq_event → action.handler → action.thread_fn → irq_finalize_oneshot lifecycle semantic equivalence: per-Documentation/core-api/genericirq.rst and per-Documentation/driver-api/basics.rst. Per-IRQF_SHARED chain semantics: per-Documentation/PCI/MSI-HOWTO.rst (PCI INTx-sharing). Per-force-threading semantics: per-Documentation/RT-patches and `force_irqthreads` cmdline behavior. Per-/proc/irq/N/smp_affinity write surface: per-Documentation/core-api/irq/irq-affinity.rst.

## Hardening

(Inherits row-1 features from `kernel/irq/00-overview.md` § Hardening.)

IRQ-manage reinforcement:

- **Per-dev_id required for IRQF_SHARED** — defense against per-shared-line free_irq picking wrong action.
- **Per-trigger / per-ONESHOT / per-PERCPU compatibility check on shared bind** — defense against per-IRQ-storm from mixed flags.
- **Per-IRQF_ONESHOT masks line until thread completes** — defense against per-level-trigger re-entry while thread runs.
- **Per-synchronize_irq inside free_irq** — defense against per-UAF freeing dev_id while handler runs.
- **Per-kthread_stop + put_task_struct on free_irq** — defense against per-kthread leak.
- **Per-IRQF_NO_THREAD / IRQF_PERCPU / IRQF_ONESHOT exempt from force-thread** — defense against per-timer-latency regression under threadirqs.
- **Per-IRQF_NO_SUSPEND lets line stay armed across suspend** — defense against per-wakeup-source loss.
- **Per-IRQF_COND_SUSPEND requires explicit cohort agreement** — defense against per-shared-line suspend-policy mismatch.
- **Per-NMI rejects threaded / shared / suspendable flags** — defense against per-NMI deadlock.
- **Per-wake_depth refcount + chip.irq_set_wake only on transitions** — defense against per-chip-write storm.
- **Per-desc.depth nonneg + WARN on enable-without-disable** — defense against per-unbalanced-enable enabling-spurious.
- **Per-affinity_notifier kref'd** — defense against per-notifier UAF.
- **Per-irq_can_set_affinity_usr gates /proc writes** — defense against per-userspace pinning interior IRQ.
- **Per-percpu request rejects non-percpu chip** — defense against per-cpu-ID confusion.
- **Per-IRQS_NMI flag on desc + handle_fasteoi_nmi path** — defense against per-NMI in normal flow.
- **Per-irq_chip_pm_get/put across request/free** — defense against per-chip-device runtime-PM drop while driven.
- **Per-IRQTF_RUNTHREAD test_and_set idempotent wake** — defense against per-double-wake of thread.
- **Per-IRQTF_READY barrier on request → thread bringup** — defense against per-IRQ-before-thread-ready.
- **Per-IRQF_NO_AUTOEN to delay auto-enable** — defense against per-spurious-on-bringup before driver init.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/irq/chip.c flow-handlers + irq_chip vtable (covered in `chip.md` Tier-3)
- kernel/irq/irqdesc.c per-desc allocator (covered in `irqdesc.md` Tier-3)
- kernel/irq/irqdomain.c hierarchical domain alloc/map (covered separately if expanded)
- kernel/irq/msi.c MSI alloc & PCI plumbing (covered separately if expanded)
- kernel/irq/pm.c suspend_device_irqs / resume_device_irqs internals (covered separately if expanded)
- kernel/irq/proc.c /proc/irq/N internal file ops (covered separately if expanded; surface registered here)
- kernel/irq/spurious.c spurious-IRQ poll (covered separately if expanded)
- kernel/irq/migration.c affinity migration (covered separately if expanded)
- arch/<arch>/kernel/irq.c per-arch dispatch (covered in per-arch Tier-2)
- drivers/irqchip/* concrete chip drivers (per-driver Tier-4)
- Implementation code
