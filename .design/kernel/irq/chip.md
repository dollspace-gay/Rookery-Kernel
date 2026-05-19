# Tier-3: kernel/irq/chip.c — IRQ-chip abstraction & flow handlers

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/irq/00-overview.md
upstream-paths:
  - kernel/irq/chip.c (~1574 lines)
  - include/linux/irq.h (struct irq_chip, irq_flow_handler_t, IRQCHIP_*)
  - include/linux/irqdesc.h
  - include/linux/msi.h (struct msi_msg)
-->

## Summary

The IRQ-chip layer is the per-`irqdesc` vtable abstraction. Per-`struct irq_chip` carries the controller-specific callbacks (irq_mask / irq_unmask / irq_ack / irq_eoi / irq_set_affinity / irq_set_type / irq_retrigger / irq_set_wake / irq_compose_msi_msg / ...) and per-`struct irq_desc.handle_irq` carries the *flow* handler (handle_level_irq, handle_edge_irq, handle_fasteoi_irq, handle_percpu_irq, handle_simple_irq, handle_nested_irq). Per-flow-handler is what `generic_handle_irq` dispatches into. Per-`chip.c` also wires the hierarchical-domain helpers (`irq_chip_*_parent`) used by stacked chips (IO-APIC → MSI → vector-domain, GICv3-ITS → its-pMSI → its-MSI, etc.). Critical for: per-controller portability (one driver path across LAPIC/IO-APIC/GIC/MSI/PIC), per-edge-vs-level semantics, per-EOI ordering, per-affinity programming, per-MSI compose.

This Tier-3 covers `kernel/irq/chip.c` (~1574 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct irq_chip` | per-controller vtable | `IrqChip` trait |
| `irq_set_chip()` | per-bind chip to irq | `Irq::set_chip` |
| `irq_set_irq_type()` | per-trigger type | `Irq::set_type` |
| `irq_set_handler_data()` | per-handler data | `Irq::set_handler_data` |
| `irq_set_chip_data()` | per-chip private data | `Irq::set_chip_data` |
| `irq_set_msi_desc_off()` / `irq_set_msi_desc()` | per-MSI desc bind | `Irq::set_msi_desc` |
| `irq_startup()` / `irq_activate()` / `irq_activate_and_startup()` | per-bring-up | `Irq::startup` / `activate` |
| `irq_shutdown()` / `irq_shutdown_and_deactivate()` | per-tear-down | `Irq::shutdown` |
| `__irq_disable()` / `irq_disable()` / `irq_enable()` | per-disable/enable path | `Irq::enable` / `disable` |
| `mask_irq()` / `unmask_irq()` / `mask_ack_irq()` / `unmask_threaded_irq()` | per-mask primitives | `Irq::mask` / `unmask` / `mask_ack` |
| `irq_percpu_enable()` / `irq_percpu_disable()` | per-CPU enable | `Irq::percpu_enable` / `percpu_disable` |
| `handle_nested_irq()` | per-nested (e.g. I2C-expander) | `IrqFlow::handle_nested` |
| `handle_simple_irq()` | per-simple flow | `IrqFlow::handle_simple` |
| `handle_untracked_irq()` | per-untracked flow | `IrqFlow::handle_untracked` |
| `handle_level_irq()` | per-level flow | `IrqFlow::handle_level` |
| `handle_fasteoi_irq()` | per-fasteoi flow | `IrqFlow::handle_fasteoi` |
| `handle_fasteoi_nmi()` | per-fasteoi-NMI flow | `IrqFlow::handle_fasteoi_nmi` |
| `handle_fasteoi_ack_irq()` / `handle_fasteoi_mask_irq()` | per-fasteoi variants | `IrqFlow::handle_fasteoi_ack` / `mask` |
| `handle_edge_irq()` | per-edge flow | `IrqFlow::handle_edge` |
| `handle_percpu_irq()` / `handle_percpu_devid_irq()` | per-CPU flow | `IrqFlow::handle_percpu` |
| `__irq_set_handler()` | per-bind flow handler | `Irq::set_flow_handler` |
| `irq_set_chained_handler_and_data()` | per-bind chained-flow | `Irq::set_chained_handler_and_data` |
| `irq_set_chip_and_handler_name()` | per-bind chip+flow | `Irq::set_chip_and_handler_name` |
| `irq_modify_status()` | per-status bit update | `Irq::modify_status` |
| `irq_cpu_online()` / `irq_cpu_offline()` | per-CPU hotplug rebalance | `Irq::cpu_online` / `cpu_offline` |
| `irq_chip_pre_redirect_parent()` | per-redirect bookkeeping | `Irq::chip_pre_redirect_parent` |
| `irq_chip_set_parent_state()` / `irq_chip_get_parent_state()` | per-hierarchy state | `Irq::chip_set_parent_state` / `get` |
| `irq_chip_shutdown_parent()` / `irq_chip_startup_parent()` | per-parent up/down | `Irq::chip_startup_parent` / `shutdown` |
| `irq_chip_enable_parent()` / `irq_chip_disable_parent()` | per-parent en/disable | `Irq::chip_enable_parent` / `disable` |
| `irq_chip_ack_parent()` / `irq_chip_mask_parent()` / `irq_chip_unmask_parent()` / `irq_chip_mask_ack_parent()` / `irq_chip_eoi_parent()` | per-parent flow primitives | `Irq::chip_*_parent` |
| `irq_chip_set_affinity_parent()` / `irq_chip_set_type_parent()` | per-parent affinity / type | `Irq::chip_set_affinity_parent` / `type` |
| `irq_chip_retrigger_hierarchy()` | per-parent retrigger walk | `Irq::chip_retrigger_hierarchy` |
| `irq_chip_set_vcpu_affinity_parent()` | per-parent vCPU affinity | `Irq::chip_set_vcpu_affinity_parent` |
| `irq_chip_set_wake_parent()` | per-parent wake | `Irq::chip_set_wake_parent` |
| `irq_chip_request_resources_parent()` / `irq_chip_release_resources_parent()` | per-parent acquire/release | `Irq::chip_request_resources_parent` / `release` |
| `irq_chip_redirect_set_affinity()` | per-redirect affinity | `Irq::chip_redirect_set_affinity` |
| `irq_chip_compose_msi_msg()` | per-MSI compose walk | `Irq::chip_compose_msi_msg` |
| `irq_chip_pm_get()` / `irq_chip_pm_put()` | per-runtime-PM ref | `Irq::chip_pm_get` / `put` |
| `irq_startup_managed()` | per-managed startup | `Irq::startup_managed` |
| `chained_action` | per-chained pseudo-action | shared global |
| `bad_chained_irq()` | per-chained sanity stub | `Irq::bad_chained_handler` |

## Compatibility contract

REQ-1: struct irq_chip vtable (subset surfaced by chip.c):
- name: per-controller human name.
- irq_startup / irq_shutdown: per-bring-up / tear-down.
- irq_enable / irq_disable: per-enable / disable (without unmask if defined).
- irq_ack: per-ack of edge-IRQ at controller.
- irq_mask / irq_mask_ack / irq_unmask: per-mask gates.
- irq_eoi: per-end-of-interrupt (fasteoi flow).
- irq_set_affinity: per-affinity-write (RC return: SET_MASK_OK / SET_MASK_OK_DONE / SET_MASK_OK_NOCOPY).
- irq_retrigger: per-fall-back software retrigger.
- irq_set_type: per-IRQF_TRIGGER_* programming.
- irq_set_wake: per-wakeup source enable.
- irq_request_resources / irq_release_resources: per-controller-allocate.
- irq_compose_msi_msg / irq_write_msi_msg: per-MSI message.
- irq_get_irqchip_state / irq_set_irqchip_state: per-controller-state introspection (PENDING / ACTIVE / MASKED).
- irq_set_vcpu_affinity: per-vCPU steering (KVM-pass-through).
- flags: IRQCHIP_SET_TYPE_MASKED / IRQCHIP_EOI_IF_HANDLED / IRQCHIP_MASK_ON_SUSPEND / IRQCHIP_ONESHOT_SAFE / IRQCHIP_AFFINITY_PRE_STARTUP / ...

REQ-2: irq_set_chip(irq, chip):
- desc = irq_get_desc_lock(irq, flags, IRQ_GET_DESC_CHECK_GLOBAL).
- if !desc: return -EINVAL.
- desc.irq_data.chip = chip ?: &no_irq_chip.
- irq_put_desc_unlock(desc, flags).
- /* Force-resume so chip gets startup on next request */
- irq_mark_irq(irq).
- return 0.

REQ-3: irq_set_irq_type(irq, type):
- desc = irq_get_desc_buslock(irq, &flags, IRQ_GET_DESC_CHECK_GLOBAL).
- if !desc: return -EINVAL.
- ret = __irq_set_trigger(desc, type).
- irq_put_desc_busunlock(desc, flags).
- return ret.

REQ-4: irq_set_handler_data(irq, data):
- desc.irq_common_data.handler_data = data.

REQ-5: irq_set_chip_data(irq, data):
- desc.irq_data.chip_data = data.

REQ-6: irq_set_msi_desc_off(base, offset, entry):
- desc = irq_get_desc_lock(base + offset, &flags, IRQ_GET_DESC_CHECK_GLOBAL).
- desc.irq_common_data.msi_desc = entry.
- if !offset: entry.irq = base.

REQ-7: irq_startup(desc, resend, force):
- /* Per-state */
- if desc.depth > 0: return IRQ_STARTUP_ABORT (else descend).
- ret = __irq_startup_managed(desc, aff, force) ∨ __irq_startup(desc).
- if resend: check_irq_resend(desc, false).
- return ret.

REQ-8: __irq_startup(desc):
- if chip.irq_startup: ret = chip.irq_startup(d).
- else: irq_enable(desc); ret = 0.
- irq_state_clr_disabled(desc); irq_state_set_started(desc).
- return ret.

REQ-9: irq_activate(desc):
- if !irqd_affinity_is_managed(d): return irq_domain_activate_irq(d, false).
- else: return 0.

REQ-10: irq_shutdown(desc):
- if irq_settings_is_started(desc):
  - desc.depth = 1.
  - if chip.irq_shutdown: chip.irq_shutdown(d); irq_state_set_disabled; irq_state_set_masked.
  - else: __irq_disable(desc, true).
- irq_state_clr_started(desc).

REQ-11: irq_shutdown_and_deactivate(desc):
- irq_shutdown(desc).
- /* irq_domain hierarchy de-activate */
- irq_domain_deactivate_irq(&desc.irq_data).

REQ-12: __irq_disable(desc, mask):
- if irqd_irq_disabled(d):
  - if mask: mask_irq(desc).
- else:
  - irq_state_set_disabled(desc).
  - if chip.irq_disable: chip.irq_disable(d); irq_state_set_masked.
  - elif mask: mask_irq(desc).

REQ-13: irq_enable(desc):
- if !irqd_irq_disabled(d): unmask_irq(desc).
- else:
  - irq_state_clr_disabled(desc).
  - if chip.irq_enable: chip.irq_enable(d); irq_state_clr_masked.
  - else: unmask_irq(desc).

REQ-14: mask_irq(desc):
- if irqd_irq_masked(d): return.
- if chip.irq_mask: chip.irq_mask(d); irq_state_set_masked(desc).

REQ-15: unmask_irq(desc):
- if !irqd_irq_masked(d): return.
- if chip.irq_unmask: chip.irq_unmask(d); irq_state_clr_masked(desc).

REQ-16: mask_ack_irq(desc):
- if chip.irq_mask_ack:
  - chip.irq_mask_ack(d); irq_state_set_masked(desc).
- else:
  - mask_irq(desc); if chip.irq_ack: chip.irq_ack(d).

REQ-17: unmask_threaded_irq(desc):
- chip = desc.irq_data.chip.
- if chip.flags & IRQCHIP_EOI_THREADED: chip.irq_eoi(&desc.irq_data).
- unmask_irq(desc).

REQ-18: handle_nested_irq(irq):
- /* No hardware flow — wakes thread directly. Used by I2C-expander GPIO. */
- desc = irq_to_desc(irq); raw_spin_lock_irq(&desc.lock).
- if !irq_can_handle_pm(desc): out_unlock.
- action = desc.action; if !action ∨ irqd_irq_disabled(&desc.irq_data): out_eoi.
- kstat_incr_irqs_this_cpu(desc).
- desc.istate |= IRQS_REPLAY; irqd_set(&desc.irq_data, IRQD_IRQ_INPROGRESS).
- raw_spin_unlock_irq.
- action_ret = IRQ_NONE.
- for action: if action.thread_fn: do_action(); action_ret |= action.handler-result.
- /* Wake threaded handler if any */
- if action_ret == IRQ_WAKE_THREAD: __irq_wake_thread(desc, action).
- raw_spin_lock_irq.
- irqd_clear(&desc.irq_data, IRQD_IRQ_INPROGRESS).
- raw_spin_unlock_irq.

REQ-19: handle_simple_irq(desc):
- /* No hw-eoi; flow handler just dispatches action. */
- raw_spin_lock(&desc.lock).
- if !irq_can_handle(desc): goto out_unlock.
- kstat_incr_irqs_this_cpu(desc).
- handle_irq_event(desc).
- raw_spin_unlock(&desc.lock).

REQ-20: handle_untracked_irq(desc):
- raw_spin_lock(&desc.lock).
- desc.istate &= ~(IRQS_REPLAY | IRQS_WAITING).
- raw_spin_unlock(&desc.lock).
- handle_irq_event_percpu(desc).
- raw_spin_lock(&desc.lock).
- raw_spin_unlock(&desc.lock).

REQ-21: handle_level_irq(desc):
- raw_spin_lock(&desc.lock).
- mask_ack_irq(desc).
- if !irq_can_handle_actions(desc): goto out_unlock.
- kstat_incr_irqs_this_cpu(desc).
- handle_irq_event(desc).
- cond_unmask_irq(desc) /* if !disabled */.
- raw_spin_unlock(&desc.lock).

REQ-22: handle_fasteoi_irq(desc):
- raw_spin_lock(&desc.lock).
- /* No ack — fasteoi controller acks internally */
- if !irq_can_handle_pm(desc):
  - if !(desc.istate & IRQS_DISABLE_UNLAZY): mask_irq(desc).
  - cond_eoi_irq(chip, &desc.irq_data); goto out.
- if irqd_has_set(&desc.irq_data, IRQD_SETAFFINITY_PENDING):
  - mask_irq(desc).
- kstat_incr_irqs_this_cpu(desc).
- if desc.istate & IRQS_ONESHOT: mask_irq(desc).
- handle_irq_event(desc).
- cond_unmask_eoi_irq(desc, chip).
- raw_spin_unlock(&desc.lock).

REQ-23: handle_fasteoi_nmi(desc):
- /* No locks — NMI context. */
- desc.irq_count++.
- action = desc.action.
- action_ret = action.handler(irq, action.dev_id).
- if chip.irq_eoi: chip.irq_eoi(&desc.irq_data).

REQ-24: handle_fasteoi_ack_irq(desc):
- raw_spin_lock(&desc.lock).
- /* fasteoi-ack: ack edge-portion, eoi end */
- if chip.irq_ack: chip.irq_ack(&desc.irq_data).
- if !irq_can_handle_actions(desc): mask_irq + eoi; goto out.
- kstat_incr_irqs_this_cpu(desc).
- if desc.istate & IRQS_ONESHOT: mask_irq(desc).
- handle_irq_event(desc).
- cond_unmask_eoi_irq(desc, chip).
- raw_spin_unlock(&desc.lock).

REQ-25: handle_fasteoi_mask_irq(desc):
- raw_spin_lock(&desc.lock).
- mask_ack_irq(desc).
- if !irq_can_handle_actions(desc): cond_eoi_irq(chip, &desc.irq_data); goto out.
- kstat_incr_irqs_this_cpu(desc).
- handle_irq_event(desc).
- cond_unmask_eoi_irq(desc, chip).
- raw_spin_unlock(&desc.lock).

REQ-26: handle_edge_irq(desc):
- raw_spin_lock(&desc.lock).
- /* Per-edge: ack first, run, may re-trigger during run */
- desc.istate &= ~(IRQS_REPLAY | IRQS_WAITING).
- if !irq_can_handle(desc):
  - desc.istate |= IRQS_PENDING; mask_ack_irq(desc); goto out_unlock.
- kstat_incr_irqs_this_cpu(desc).
- chip.irq_ack(&desc.irq_data).
- do:
  - if unlikely(!desc.action):
    - mask_irq(desc); break.
  - /* Pending re-trigger arrived while handler ran */
  - if unlikely(desc.istate & IRQS_PENDING):
    - if !irqd_irq_disabled(&desc.irq_data) ∧ irqd_irq_masked(&desc.irq_data): unmask_irq(desc).
  - handle_irq_event(desc).
- while (desc.istate & IRQS_PENDING) ∧ !irqd_irq_disabled(&desc.irq_data).
- raw_spin_unlock(&desc.lock).

REQ-27: handle_percpu_irq(desc):
- /* No locks: per-CPU IRQ */
- kstat_incr_irqs_this_cpu(desc).
- if chip.irq_ack: chip.irq_ack(&desc.irq_data).
- handle_irq_event_percpu(desc).
- if chip.irq_eoi: chip.irq_eoi(&desc.irq_data).

REQ-28: handle_percpu_devid_irq(desc):
- /* Per-CPU devid: dev_id is __percpu */
- action = desc.action.
- kstat_incr_irqs_this_cpu(desc).
- if chip.irq_ack: chip.irq_ack(&desc.irq_data).
- if action: res = action.handler(irq, raw_cpu_ptr(action.percpu_dev_id)); WARN if !IRQ_HANDLED.
- if chip.irq_eoi: chip.irq_eoi(&desc.irq_data).

REQ-29: __irq_set_handler(irq, handle, is_chained, name):
- desc = irq_get_desc_buslock(irq, &flags, IRQ_GET_DESC_CHECK_GLOBAL).
- __irq_do_set_handler(desc, handle, is_chained, name).
- irq_put_desc_busunlock(desc, flags).

REQ-30: __irq_do_set_handler(desc, handle, is_chained, name):
- /* Replace flow handler. */
- if !handle: handle = handle_bad_irq; chip = &no_irq_chip.
- else if WARN_ON(!desc.irq_data.chip ∨ desc.irq_data.chip == &no_irq_chip): goto out.
- if is_chained:
  - WARN_ON(handle == handle_bad_irq).
  - irq_settings_set_noprobe(desc).
  - irq_settings_set_norequest(desc).
  - irq_settings_set_nothread(desc).
  - desc.action = &chained_action.
  - irq_activate_and_startup(desc, IRQ_RESEND).
- desc.handle_irq = handle.
- desc.name = name.

REQ-31: irq_set_chained_handler_and_data(irq, handle, data):
- desc = irq_get_desc_buslock(irq, &flags, 0).
- desc.irq_common_data.handler_data = data.
- __irq_do_set_handler(desc, handle, 1, NULL).

REQ-32: irq_modify_status(irq, clr, set):
- desc.status_use_accessors &= ~clr.
- desc.status_use_accessors |= set.
- /* irqd flags re-derived */

REQ-33: irq_cpu_online() / irq_cpu_offline():
- Walk all irq_desc; if affinity includes hot-CPU:
  - online: chip.irq_cpu_online(d).
  - offline: chip.irq_cpu_offline(d).

REQ-34: chained_action / bad_chained_irq:
- chained_action: { .handler = bad_chained_irq, .name = "chained-irq" } — sentinel "do not request_irq this; it is the chained-flow target".
- bad_chained_irq: WARN_ONCE; returns IRQ_NONE.

REQ-35: Per-hierarchical chip helpers (irq_chip_*_parent):
- All walk irq_data → parent_data chain.
- irq_chip_ack_parent / mask / unmask / mask_ack / eoi: invoke chip.irq_*(parent_data).
- irq_chip_set_affinity_parent(d, mask, force): if parent: parent.chip.irq_set_affinity(parent, mask, force); else: -ENOSYS.
- irq_chip_set_type_parent(d, type): parent.chip.irq_set_type(parent, type).
- irq_chip_retrigger_hierarchy(d): while data: if data.chip.irq_retrigger: return data.chip.irq_retrigger(data); data = data.parent_data; return 0.
- irq_chip_set_vcpu_affinity_parent(d, info): parent.chip.irq_set_vcpu_affinity(parent, info).
- irq_chip_set_wake_parent(d, on): parent.chip.irq_set_wake(parent, on).
- irq_chip_request_resources_parent / release_resources_parent: walk to parent that implements.
- irq_chip_compose_msi_msg(d, msg): walk to first ancestor with irq_compose_msi_msg; invoke.

REQ-36: irq_chip_set_parent_state(d, which, val) / get_parent_state:
- /* For irqd_set_irqchip_state on stacked chips */
- /* Walks to first parent with chip.irq_set_irqchip_state / get */
- if !parent.chip.irq_set_irqchip_state: return -EINVAL.
- return parent.chip.irq_set_irqchip_state(parent, which, val).

REQ-37: irq_chip_pre_redirect_parent(d):
- /* Prepares hierarchy for affinity-redirect (vector-domain pre-step) */
- if parent.chip.irq_pre_redirect_parent: parent.chip.irq_pre_redirect_parent(parent).

REQ-38: irq_chip_pm_get(d) / irq_chip_pm_put(d):
- /* Runtime-PM refcount the IRQ-chip device */
- dev = irq_get_pm_device(data).
- if dev: pm_runtime_get_sync(dev) / pm_runtime_put(dev).

REQ-39: cond_unmask_irq(desc): unmask only if !disabled ∧ !ONESHOT-or-handler-finished.
- if irqd_irq_disabled(&desc.irq_data): return.
- if desc.istate & IRQS_ONESHOT: return.
- unmask_irq(desc).

REQ-40: cond_unmask_eoi_irq(desc, chip):
- if !(chip.flags & IRQCHIP_EOI_IF_HANDLED) ∨ !(desc.istate & IRQS_ONESHOT):
  - chip.irq_eoi(&desc.irq_data).
- cond_unmask_irq(desc).

REQ-41: cond_eoi_irq(chip, data):
- if !(chip.flags & IRQCHIP_EOI_IF_HANDLED): chip.irq_eoi(data).

REQ-42: irq_can_handle / irq_can_handle_pm / irq_can_handle_actions:
- pm: !suspended in pm-cycle.
- actions: desc.action != NULL.
- can_handle: pm ∧ actions ∧ !disabled.

REQ-43: irq_startup_managed(desc):
- /* Affinity-managed IRQ */
- if irqd_affinity_is_managed(d) ∧ irqd_affinity_is_managed_and_shutdown(d):
  - irq_state_clr_managed_shutdown(d).
- /* Choose best CPU from mask */

REQ-44: irq_set_chip_and_handler_name(irq, chip, handle, name):
- irq_set_chip(irq, chip); __irq_set_handler(irq, handle, 0, name).

REQ-45: irq_percpu_enable / irq_percpu_disable:
- /* Per-CPU IRQ: per-cpu enable_count */
- if chip.irq_enable / irq_disable: invoke; else: unmask / mask.

## Acceptance Criteria

- [ ] AC-1: irq_set_chip binds irq_chip vtable; NULL becomes &no_irq_chip.
- [ ] AC-2: handle_level_irq: mask_ack before handler; unmask after if !disabled.
- [ ] AC-3: handle_edge_irq: ack first; re-enters handler while IRQS_PENDING.
- [ ] AC-4: handle_fasteoi_irq: no ack; eoi after handler unless ONESHOT.
- [ ] AC-5: handle_percpu_irq: no spinlock; per-CPU stat increment.
- [ ] AC-6: handle_nested_irq: no hw flow; runs threaded handler in calling thread context.
- [ ] AC-7: handle_simple_irq: no mask/ack/eoi; flow does action only.
- [ ] AC-8: irq_chip_set_parent_state walks to first parent with .irq_set_irqchip_state.
- [ ] AC-9: irq_chip_compose_msi_msg returns first ancestor's compose result.
- [ ] AC-10: irq_chip_*_parent return -ENOSYS if no parent implements.
- [ ] AC-11: irq_chip_request_resources_parent calls parent.chip.irq_request_resources.
- [ ] AC-12: chained_action's handler is bad_chained_irq sentinel.
- [ ] AC-13: irq_startup invokes chip.irq_startup if set else irq_enable.
- [ ] AC-14: irq_shutdown calls chip.irq_shutdown if set else __irq_disable.
- [ ] AC-15: handle_fasteoi_nmi runs without locking.
- [ ] AC-16: irq_chip_pm_get/put balances pm-runtime refcount.

## Architecture

```
trait IrqChip {
  fn name(&self) -> &str;
  fn startup(&self, d: &IrqData) -> i32 { 0 }
  fn shutdown(&self, d: &IrqData) {}
  fn enable(&self, d: &IrqData) {}
  fn disable(&self, d: &IrqData) {}
  fn ack(&self, d: &IrqData) {}
  fn mask(&self, d: &IrqData) {}
  fn mask_ack(&self, d: &IrqData) {}
  fn unmask(&self, d: &IrqData) {}
  fn eoi(&self, d: &IrqData) {}
  fn set_affinity(&self, d: &IrqData, m: &Cpumask, force: bool) -> Result<SetAffinityOk, Errno> { Err(Errno::ENOSYS) }
  fn retrigger(&self, d: &IrqData) -> bool { false }
  fn set_type(&self, d: &IrqData, ty: u32) -> Result<(), Errno> { Err(Errno::ENOSYS) }
  fn set_wake(&self, d: &IrqData, on: u32) -> Result<(), Errno> { Err(Errno::ENOSYS) }
  fn request_resources(&self, d: &IrqData) -> Result<(), Errno> { Ok(()) }
  fn release_resources(&self, d: &IrqData) {}
  fn compose_msi_msg(&self, d: &IrqData, m: &mut MsiMsg) {}
  fn write_msi_msg(&self, d: &IrqData, m: &MsiMsg) {}
  fn get_irqchip_state(&self, d: &IrqData, w: IrqState) -> Result<bool, Errno> { Err(Errno::ENOSYS) }
  fn set_irqchip_state(&self, d: &IrqData, w: IrqState, v: bool) -> Result<(), Errno> { Err(Errno::ENOSYS) }
  fn set_vcpu_affinity(&self, d: &IrqData, info: *mut ()) -> Result<(), Errno> { Err(Errno::ENOSYS) }
  fn cpu_online(&self, d: &IrqData) {}
  fn cpu_offline(&self, d: &IrqData) {}
  fn flags(&self) -> u32 { 0 }       // IRQCHIP_*
}
```

`Irq::set_chip(irq, chip) -> Result<(), Errno>`:
1. desc = irq_get_desc_lock(irq, IRQ_GET_DESC_CHECK_GLOBAL)?.
2. desc.irq_data.chip = chip.unwrap_or(&NO_IRQ_CHIP).
3. irq_put_desc_unlock(desc).
4. Irq::mark(irq).

`IrqFlow::handle_level(desc)`:
1. raw_spin_lock(&desc.lock).
2. mask_ack_irq(desc).
3. if !irq_can_handle_actions(desc): goto out_unlock.
4. kstat_incr_irqs_this_cpu(desc).
5. handle_irq_event(desc) /* runs action chain */.
6. cond_unmask_irq(desc).
7. raw_spin_unlock(&desc.lock).

`IrqFlow::handle_edge(desc)`:
1. raw_spin_lock(&desc.lock).
2. desc.istate &= !(IRQS_REPLAY | IRQS_WAITING).
3. if !irq_can_handle(desc):
   - desc.istate |= IRQS_PENDING; mask_ack_irq(desc); goto out_unlock.
4. kstat_incr_irqs_this_cpu(desc).
5. desc.chip.ack(&desc.irq_data).
6. loop:
   - if desc.action.is_none(): mask_irq(desc); break.
   - if desc.istate & IRQS_PENDING ∧ !disabled ∧ masked: unmask_irq(desc).
   - handle_irq_event(desc).
   - if !(desc.istate & IRQS_PENDING): break.
   - if irqd_irq_disabled(&desc.irq_data): break.
7. raw_spin_unlock(&desc.lock).

`IrqFlow::handle_fasteoi(desc)`:
1. raw_spin_lock(&desc.lock).
2. chip = desc.irq_data.chip.
3. if !irq_can_handle_pm(desc):
   - if !(desc.istate & IRQS_DISABLE_UNLAZY): mask_irq(desc).
   - cond_eoi_irq(chip, &desc.irq_data); goto out.
4. if irqd_has_set(&desc.irq_data, IRQD_SETAFFINITY_PENDING): mask_irq(desc).
5. kstat_incr_irqs_this_cpu(desc).
6. if desc.istate & IRQS_ONESHOT: mask_irq(desc).
7. handle_irq_event(desc).
8. cond_unmask_eoi_irq(desc, chip).
9. raw_spin_unlock(&desc.lock).

`IrqFlow::handle_percpu(desc)`:
1. kstat_incr_irqs_this_cpu(desc).
2. desc.chip.ack(&desc.irq_data) (if-defined).
3. handle_irq_event_percpu(desc).
4. desc.chip.eoi(&desc.irq_data) (if-defined).

`IrqFlow::handle_simple(desc)`:
1. raw_spin_lock(&desc.lock).
2. if !irq_can_handle(desc): goto out_unlock.
3. kstat_incr_irqs_this_cpu(desc).
4. handle_irq_event(desc).
5. raw_spin_unlock(&desc.lock).

`IrqFlow::handle_nested(irq)`:
1. desc = irq_to_desc(irq).
2. raw_spin_lock_irq(&desc.lock).
3. if !irq_can_handle_pm(desc): goto out_unlock.
4. action = desc.action; if !action ∨ disabled: goto out_eoi.
5. kstat_incr_irqs_this_cpu(desc).
6. irqd_set(&desc.irq_data, IRQD_IRQ_INPROGRESS).
7. raw_spin_unlock_irq.
8. for a in actions: a.thread_fn(irq, a.dev_id) /* in caller thread context */.
9. raw_spin_lock_irq.
10. irqd_clear(&desc.irq_data, IRQD_IRQ_INPROGRESS).
11. raw_spin_unlock_irq.

`Irq::chip_compose_msi_msg(d, msg) -> Result<(), Errno>`:
1. while data:
   - if data.chip.irq_compose_msi_msg:
     - data.chip.compose_msi_msg(data, msg). return Ok(()).
   - data = data.parent_data.
2. return Err(Errno::ENOSYS).

`Irq::chip_set_parent_state(d, which, val) -> Result<(), Errno>`:
1. data = d.parent_data; while data:
   - if data.chip.irq_set_irqchip_state:
     - return data.chip.set_irqchip_state(data, which, val).
   - data = data.parent_data.
2. return Err(Errno::EINVAL).

`Irq::chip_retrigger_hierarchy(d) -> bool`:
1. data = d; while data:
   - if data.chip.irq_retrigger:
     - return data.chip.retrigger(data).
   - data = data.parent_data.
2. false.

`Irq::chip_request_resources_parent(d) -> Result<(), Errno>`:
1. data = d.parent_data; while data:
   - if data.chip.irq_request_resources:
     - return data.chip.request_resources(data).
   - data = data.parent_data.
2. Err(Errno::ENOSYS).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `chip_lock_held_in_flow` | INVARIANT | per-handle_{level,edge,fasteoi,simple,nested}: desc.lock held during action dispatch (except percpu/nmi). |
| `edge_pending_loop_terminates` | INVARIANT | per-handle_edge: loops while IRQS_PENDING ∧ !disabled — finite by IRQS_PENDING set/clear monotonicity per cycle. |
| `fasteoi_eoi_balanced` | INVARIANT | per-handle_fasteoi: every cond_eoi_irq / cond_unmask_eoi_irq matches exactly one chip.irq_eoi. |
| `percpu_no_spinlock` | INVARIANT | per-handle_percpu / handle_percpu_devid: never acquires desc.lock. |
| `nmi_no_spinlock` | INVARIANT | per-handle_fasteoi_nmi: never acquires desc.lock. |
| `chained_action_immutable` | INVARIANT | per-chained_action: never request_irq'd; bad_chained_irq returns IRQ_NONE. |
| `compose_msi_first_ancestor` | INVARIANT | per-irq_chip_compose_msi_msg: returns 0 only when first-implementing ancestor found. |
| `parent_state_walk_terminates` | INVARIANT | per-irq_chip_set_parent_state: walks finite parent_data chain. |
| `pm_get_put_balanced` | INVARIANT | per-irq_chip_pm_get / irq_chip_pm_put: 1:1 refcount. |
| `set_chip_chip_data_only_in_lock` | INVARIANT | per-irq_set_chip / set_chip_data / set_handler_data: desc-lock held. |

### Layer 2: TLA+

`kernel/irq/chip.tla`:
- Per-flow-handler state machine (LEVEL / EDGE / FASTEOI / FASTEOI_ACK / FASTEOI_MASK / SIMPLE / PERCPU / PERCPU_DEVID / NESTED / FASTEOI_NMI).
- Per-irq lifecycle: STARTUP → ACTIVE → SHUTDOWN.
- Properties:
  - `safety_level_unmask_after_handler` — per-level: unmask only when !disabled.
  - `safety_edge_ack_before_handler` — per-edge: chip.ack precedes any handler call this cycle.
  - `safety_fasteoi_oneshot_implies_mask` — per-fasteoi: IRQS_ONESHOT ⟹ mask_irq before handler.
  - `safety_fasteoi_eoi_required` — per-fasteoi: chip.eoi reached on every termination.
  - `safety_percpu_no_lock` — per-percpu: lock not held.
  - `liveness_pending_reentry_drains` — per-edge: every IRQS_PENDING eventually consumed or masked.
  - `liveness_set_chip_visible_to_set_handler` — per-cross-API ordering: set_chip happens-before set_handler.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Irq::set_chip` post: desc.irq_data.chip = chip ∨ NO_IRQ_CHIP | `Irq::set_chip` |
| `IrqFlow::handle_level` post: mask_ack ran iff IRQS_PENDING-not-set; unmask iff !disabled ∧ !ONESHOT | `IrqFlow::handle_level` |
| `IrqFlow::handle_edge` post: chip.ack ran ≥ 1; final state masked iff IRQS_PENDING ∨ disabled | `IrqFlow::handle_edge` |
| `IrqFlow::handle_fasteoi` post: chip.eoi ran exactly once iff !(EOI_IF_HANDLED ∧ ONESHOT-not-completed) | `IrqFlow::handle_fasteoi` |
| `IrqFlow::handle_percpu` post: chip.ack and chip.eoi run if defined; per-cpu kstat incremented | `IrqFlow::handle_percpu` |
| `IrqFlow::handle_nested` post: action.thread_fn invoked in calling-task context | `IrqFlow::handle_nested` |
| `Irq::chip_compose_msi_msg` post: msg filled by first ancestor with .irq_compose_msi_msg | `Irq::chip_compose_msi_msg` |
| `Irq::chip_set_parent_state` post: returns first parent's set_irqchip_state result or EINVAL | `Irq::chip_set_parent_state` |
| `Irq::chip_request_resources_parent` post: returns first-implementing-ancestor result or ENOSYS | `Irq::chip_request_resources_parent` |
| `Irq::chip_pm_get` post: pm_runtime ref incremented iff chip-device | `Irq::chip_pm_get` |

### Layer 4: Verus/Creusot functional

Per-flow `chip.ack / chip.eoi / chip.mask / chip.unmask` ordering semantic equivalence vs upstream `kernel/irq/chip.c`: per-Documentation/core-api/genericirq.rst and per-table at chip.c:686-915 commentary. Per-hierarchical-chip `irq_chip_*_parent` walk equivalence: per-Documentation/core-api/irq/irq-domain.rst hierarchical-IRQ-domain semantics. Per-handle_nested vs handle_simple comparison: per-Documentation/driver-api/gpio/driver.rst nested-vs-simple threaded GPIO controllers.

## Hardening

(Inherits row-1 features from `kernel/irq/00-overview.md` § Hardening.)

IRQ-chip reinforcement:

- **Per-NO_IRQ_CHIP default for NULL chip** — defense against per-null-vtable deref on flow dispatch.
- **Per-chained_action sentinel + bad_chained_irq WARN_ONCE** — defense against per-double-bind on chained-flow line.
- **Per-edge handler IRQS_PENDING bounded by mask** — defense against per-livelock (storming edge keeps re-entering).
- **Per-fasteoi EOI never skipped on early-return** — defense against per-controller-stuck-line.
- **Per-percpu/nmi-flow no spinlock** — defense against per-NMI/percpu reentrancy deadlock.
- **Per-IRQCHIP_EOI_THREADED gates threaded unmask** — defense against per-threaded-eoi inversion.
- **Per-hierarchical walk bounded by parent_data nullability** — defense against per-infinite-loop in malformed hierarchy.
- **Per-pm_get/put refcounting** — defense against per-chip-device runtime-PM drop while-driven.
- **Per-irq_chip_pre_redirect_parent before affinity** — defense against per-vector-leak on redirect.
- **Per-WARN_ON !chip || chip==&no_irq_chip on set_handler** — defense against per-handler-without-chip.
- **Per-irq_settings_set_noprobe/norequest/nothread on chained-bind** — defense against per-userspace-request hijack of chained line.
- **Per-IRQCHIP_AFFINITY_PRE_STARTUP gating** — defense against per-affinity-write before vector allocation.
- **Per-chip.irq_request_resources mandatory acquire before startup** — defense against per-shared-resource conflict (GPIO-pinmux).

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — copy-out of irq_chip/effective-affinity state through /proc/interrupts is bounds-checked.
- **PAX_KERNEXEC** — every `struct irq_chip` instance is const and W^X; no writable irq_chip vtables anywhere in the build.
- **PAX_RANDKSTACK** — randomize kernel stack on every interrupt entry that dispatches through `handle_*_irq`.
- **PAX_REFCOUNT** — saturating atomics on irq_chip parent-data refs, msi_domain refs, and chained-handler installs.
- **PAX_MEMORY_SANITIZE** — scrub freed `irq_data`/`irq_common_data` allocations on hierarchy teardown to deny stale parent pointers from being reused.
- **PAX_UDEREF** — user/kernel pointer split rigorously preserved on the chained-handler fast path (no user deref from irq context).
- **PAX_RAP / kCFI** — type-signed indirect calls for *every* `irq_chip` op (`irq_mask`, `irq_unmask`, `irq_eoi`, `irq_set_affinity`, `irq_set_type`, `irq_request_resources`, hierarchical alloc/free).
- **GRKERNSEC_HIDESYM** — hide `irq_default_chip`, `no_irq_chip`, `dummy_irq_chip`, and per-arch IO-APIC/MSI chips from /proc/kallsyms.
- **GRKERNSEC_DMESG** — restrict "Unknown IRQ chip"/EOI warnings to CAP_SYSLOG.
- **irq_chip vtable PAX_RAP** — handle_level_irq, handle_edge_irq, handle_fasteoi_irq, handle_simple_irq, and handle_percpu_irq dispatch through type-signed indirect call wrappers; a forged irq_chip cannot redirect dispatch to arbitrary kernel text.
- **handle_*_irq dispatch hardened** — each handler validates `desc->handler_data` and `irq_desc_get_chip(desc) != &no_irq_chip` before fanning out; mismatched flow handlers are rejected with `WARN_ON` instead of silently dispatching.
- **Chained handler install gated** — `irq_set_chained_handler*` forces `IRQ_NOREQUEST | IRQ_NOPROBE | IRQ_NOTHREAD` so userspace cannot later claim or steer the chained line.
- **Hierarchical walk bounded** — `irq_chip_*_parent` walks stop on `data->parent_data == NULL`, and the maximum hierarchy depth is checked at domain registration; malformed hierarchies cannot loop the EOI path.
- **PM acquire under refcount** — `pm_runtime_get` on chip device uses PAX_REFCOUNT; the chip cannot suspend mid-IRQ-flow.
- **Rationale**: irq_chip is the canonical indirect-call cluster in the kernel — every line dispatches through five or more vtable slots per IRQ. PAX_RAP on the entire op set plus const W^X chip instances eliminates the "forged chip via slab UAF" class and keeps the IRQ flow handlers genuinely typed.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/irq/manage.c request_irq / threaded-handler runtime (covered in `manage.md` Tier-3)
- kernel/irq/irqdesc.c per-desc allocator (covered in `irqdesc.md` Tier-3)
- kernel/irq/irqdomain.c hierarchical domain alloc/map (covered separately if expanded)
- kernel/irq/msi.c MSI alloc & PCI plumbing (covered separately if expanded)
- kernel/irq/proc.c /proc/irq/N (covered in `manage.md` Tier-3 § /proc)
- arch/x86/kernel/apic/* concrete irq_chip instances (covered in `apic.md` if expanded)
- drivers/irqchip/* concrete irq_chip instances (per-driver Tier-4)
- Implementation code
