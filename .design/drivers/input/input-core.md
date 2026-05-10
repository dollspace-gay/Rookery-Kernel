# Tier-3: drivers/input/input.c — Input subsystem core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/input/00-overview.md
upstream-paths:
  - drivers/input/input.c (~2770 lines)
  - drivers/input/input-core-private.h
  - drivers/input/input-compat.h
  - drivers/input/input-poller.h
  - include/linux/input.h
  - include/uapi/linux/input.h
  - include/uapi/linux/input-event-codes.h
-->

## Summary

The **input core** is the broker between input device drivers (keyboards, mice, joysticks, touchscreens, switches, sensors) and event consumers (`evdev`, `mousedev`, `joydev`, `kbd`, `sysrq`). Per-`struct input_dev` describes a hardware source with per-event-type capability bitmaps (`evbit`, `keybit`, `relbit`, `absbit`, `mscbit`, `swbit`, `ledbit`, `sndbit`, `ffbit`, `propbit`). Per-`struct input_handler` describes an event consumer with an `id_table` matching rule and `connect/disconnect/event/events/filter` vtable. Per-`struct input_handle` is the M:N edge linking a single (dev, handler) pair, threaded onto both `dev->h_list` (filters head, regular tail) and `handler->h_list`. Per-`input_event()` enters under `dev->event_lock` (IRQ-safe), passes through `input_get_disposition()` (type dispatch + state-bit update + abs defuzz) and `input_event_dispose()` which batches per-`input_value` triplets in `dev->vals[]` until `SYN_REPORT` flushes through `input_pass_values()`, which RCU-walks `dev->h_list` honoring a single grab via `dev->grab`. Per-EV_REP autorepeat: `input_repeat_key` timer re-injects value=2 events at REP_PERIOD after REP_DELAY ms. Per-`input_inject_event()` lets handlers feed events back into the pipeline subject to grab arbitration. Critical for: ABI-compatible `/dev/input/eventN` event delivery, sysrq, console keyboard, virtual-console-switch, suspend/resume key release.

This Tier-3 covers `drivers/input/input.c` (~2770 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct input_dev` | per-device descriptor + cap bitmaps | `InputDev` |
| `struct input_handler` | per-consumer vtable + id-table | `InputHandler` |
| `struct input_handle` | per-(dev,handler) edge | `InputHandle` |
| `struct input_value` | per-event triplet (type, code, value) | `InputValue` |
| `input_event()` | per-driver event entry | `Input::event` |
| `input_handle_event()` | per-locked dispatch | `Input::handle_event` |
| `input_get_disposition()` | per-type dispatch + state update | `Input::get_disposition` |
| `input_event_dispose()` | per-batch + flush | `Input::event_dispose` |
| `input_pass_values()` | per-handle walk | `Input::pass_values` |
| `input_inject_event()` | per-handler back-feed | `Input::inject_event` |
| `input_handle_abs_event()` | per-EV_ABS defuzz + MT slot | `Input::handle_abs_event` |
| `input_defuzz_abs_event()` | per-axis fuzz filter | `Input::defuzz_abs_event` |
| `input_start_autorepeat()` / `_stop_autorepeat()` | per-EV_REP timer arm/disarm | `Input::start_autorepeat` / `stop_autorepeat` |
| `input_repeat_key()` | per-EV_REP timer callback | `Input::repeat_key` |
| `input_enable_softrepeat()` | per-set REP_DELAY/REP_PERIOD | `Input::enable_softrepeat` |
| `input_allocate_device()` / `devm_input_allocate_device()` | per-alloc | `Input::allocate_device` / `devm_allocate_device` |
| `input_free_device()` | per-free unregistered | `Input::free_device` |
| `input_register_device()` | per-register + attach handlers | `Input::register_device` |
| `input_unregister_device()` | per-unregister + disconnect handles | `Input::unregister_device` |
| `__input_unregister_device()` | per-internal teardown | `Input::unregister_device_inner` |
| `input_register_handler()` / `_unregister_handler()` | per-handler lifecycle | `Input::register_handler` / `unregister_handler` |
| `input_register_handle()` / `_unregister_handle()` | per-edge lifecycle | `Input::register_handle` / `unregister_handle` |
| `input_attach_handler()` | per-id-table match + connect | `Input::attach_handler` |
| `input_match_device()` / `_match_device_id()` | per-id-table walk | `Input::match_device` / `match_device_id` |
| `input_grab_device()` / `_release_device()` | per-exclusive ownership | `Input::grab_device` / `release_device` |
| `input_open_device()` / `_close_device()` / `_flush_device()` | per-handle open/close/flush | `Input::open_device` / `close_device` / `flush_device` |
| `input_reset_device()` | per-resume key-release + LED restore | `Input::reset_device` |
| `input_inhibit_device()` / `_uninhibit_device()` | per-sysfs inhibit | `Input::inhibit_device` / `uninhibit_device` |
| `input_dev_release_keys()` | per-suspend/disconnect simulate-keyup | `Input::release_keys` |
| `input_disconnect_device()` | per-going-away serialize | `Input::disconnect_device` |
| `input_set_capability()` | per-cap-bit set + EV-promote | `Input::set_capability` |
| `input_set_abs_params()` / `_alloc_absinfo()` / `_copy_abs()` | per-EV_ABS init | `Input::set_abs_params` / `alloc_absinfo` / `copy_abs` |
| `input_set_keycode()` / `_get_keycode()` | per-keymap mutate/read | `Input::set_keycode` / `get_keycode` |
| `input_default_setkeycode()` / `_default_getkeycode()` | per-default keymap vtable | `Input::default_setkeycode` / `default_getkeycode` |
| `input_scancode_to_scalar()` | per-keymap-entry scalarize | `Input::scancode_to_scalar` |
| `input_fetch_keycode()` | per-keycodesize fetch | `Input::fetch_keycode` |
| `input_set_timestamp()` / `_get_timestamp()` | per-event timestamp (MONO/REAL/BOOT) | `Input::set_timestamp` / `get_timestamp` |
| `input_estimate_events_per_packet()` | per-vals[] sizing | `Input::estimate_events_per_packet` |
| `input_device_tune_vals()` | per-register vals[] resize | `Input::device_tune_vals` |
| `input_cleanse_bitmasks()` | per-register zero-unannounced bitmap | `Input::cleanse_bitmasks` |
| `input_dev_toggle()` | per-suspend/resume LED+SND+REP restore | `Input::dev_toggle` |
| `input_get_new_minor()` / `_free_minor()` | per-input-major IDA | `Input::get_new_minor` / `free_minor` |
| `input_handler_for_each_handle()` | per-handler RCU iter | `Input::handler_for_each_handle` |
| `input_handle_setup_event_handler()` | per-pick handle_events fn | `Input::setup_event_handler` |
| `input_class` | per-sysfs class "input" | `Input::class` |
| `input_dev_attr_groups[]` | per-sysfs attrs (name/phys/uniq/modalias/properties/inhibited/id/capabilities) | `Input::dev_attr_groups` |
| `input_dev_uevent()` | per-modalias hotplug | `Input::dev_uevent` |
| `input_dev_pm_ops` | per-suspend/resume/freeze/poweroff | `Input::dev_pm_ops` |

## Compatibility contract

REQ-1: struct input_dev fields:
- name, phys, uniq: identification strings.
- id: { bustype, vendor, product, version } u16 quartet.
- propbit[BITS_TO_LONGS(INPUT_PROP_MAX)]: device property bits.
- evbit[BITS_TO_LONGS(EV_MAX)]: type bits (EV_SYN/KEY/REL/ABS/MSC/SW/LED/SND/REP/FF/PWR/FF_STATUS).
- keybit, relbit, absbit, mscbit, swbit, ledbit, sndbit, ffbit: per-type code bitmaps.
- absinfo: kmalloc'd array of struct input_absinfo[ABS_CNT] (only if EV_ABS).
- key, led, snd, sw: current state bitmaps (toggle-tracked).
- rep[REP_CNT]: { REP_DELAY, REP_PERIOD } ms.
- vals: kmalloc'd array of struct input_value, length max_vals; batch buffer between flushes.
- num_vals: current count.
- max_vals: capacity.
- hint_events_per_packet: driver-declared estimate (post-tune ≥ estimate).
- mt: optional struct input_mt (multitouch slots).
- timer: struct timer_list for software autorepeat.
- repeat_key: code currently auto-repeating.
- timestamp[INPUT_CLK_MAX]: { MONO, REAL, BOOT } ktimes.
- dev: embedded struct device.
- h_list: list of input_handle (filters head, handlers tail).
- node: input_dev_list linkage.
- mutex: serializes open/close/grab.
- event_lock: spinlock_irqsave; protects state bitmaps + vals[].
- grab: RCU-protected handle * for exclusive owner.
- going_away, inhibited, users, devres_managed: state flags + refcounts.
- open, close, flush, event, getkeycode, setkeycode: driver vtable callbacks.

REQ-2: struct input_handler fields:
- name.
- filter, event, events: mutually exclusive event delivery callbacks (only one may be non-NULL — enforced by input_handler_check_methods()).
- match: optional finer-grained matcher beyond id_table.
- connect, disconnect: handle lifecycle callbacks.
- start: post-grab-release / post-open synchronize callback.
- legacy_minors: bool.
- legacy_base, minor: legacy minor range.
- passive_observer: bool — handle does not affect dev->users refcount on open.
- id_table: NULL-terminated struct input_device_id array.
- h_list: list of input_handle.
- node: input_handler_list linkage.

REQ-3: struct input_handle fields:
- private: handler-defined pointer.
- name.
- open: refcount of opens through this handle.
- dev: input_dev *.
- handler: input_handler *.
- d_node: dev->h_list linkage.
- h_node: handler->h_list linkage.
- handle_events: function pointer set by input_handle_setup_event_handler() to one of:
  - input_handle_events_filter (if handler->filter).
  - input_handle_events_default (if handler->event).
  - input_handle_events_null (no callback).
  - handler->events (if handler->events).

REQ-4: input_event(dev, type, code, value):
- if !is_event_supported(type, dev.evbit, EV_MAX): return.
- guard(spinlock_irqsave)(&dev.event_lock).
- input_handle_event(dev, type, code, value).

REQ-5: input_handle_event:
- lockdep_assert_held(&dev.event_lock).
- disposition = input_get_disposition(dev, type, code, &value).
- if disposition != INPUT_IGNORE_EVENT:
  - if type != EV_SYN: add_input_randomness(type, code, value).
  - input_event_dispose(dev, disposition, type, code, value).

REQ-6: input_get_disposition per-type semantics:
- if dev.inhibited: return INPUT_IGNORE_EVENT.
- EV_SYN/SYN_CONFIG: PASS_TO_ALL.
- EV_SYN/SYN_REPORT: PASS_TO_HANDLERS | FLUSH.
- EV_SYN/SYN_MT_REPORT: PASS_TO_HANDLERS.
- EV_KEY: only if code supported in keybit; if value == 2 (repeat) → PASS_TO_HANDLERS bypass state update; else only if state-bit changes — update dev.key and PASS_TO_HANDLERS (key-up→key-down or vice versa, ignore stuck-state events).
- EV_SW: only if supported ∧ sw-bit changes — update dev.sw and PASS_TO_HANDLERS.
- EV_ABS: input_handle_abs_event() — ABS_MT_SLOT stages slot; multitouch MT-axes write per-slot; non-MT writes absinfo[code].value; defuzz; if unchanged ignore; if new MT slot also emit ABS_MT_SLOT (INPUT_SLOT flag).
- EV_REL: only if supported ∧ value != 0 — PASS_TO_HANDLERS.
- EV_MSC: only if supported — PASS_TO_ALL.
- EV_LED: only if supported ∧ led-bit changes — update dev.led and PASS_TO_ALL.
- EV_SND: only if supported — toggle if value changed, PASS_TO_ALL.
- EV_REP: if code ≤ REP_MAX ∧ value ≥ 0 ∧ rep[code] changes — update rep, PASS_TO_ALL.
- EV_FF: if value ≥ 0 — PASS_TO_ALL.
- EV_PWR: PASS_TO_ALL.

REQ-7: input_handle_abs_event:
- if code == ABS_MT_SLOT: stage mt.slot = value; return INPUT_IGNORE_EVENT.
- is_mt_event = input_is_mt_value(code).
- if !is_mt_event: pold = &absinfo[code].value.
- else if mt: pold = &mt.slots[mt.slot].abs[code - ABS_MT_FIRST]; is_new_slot = (mt.slot != absinfo[ABS_MT_SLOT].value).
- else: pold = NULL (bypass filtering when no slots).
- if pold: value = defuzz(value, *pold, absinfo[code].fuzz); if unchanged → INPUT_IGNORE_EVENT; else *pold = value.
- if is_new_slot: absinfo[ABS_MT_SLOT].value = mt.slot; return PASS_TO_HANDLERS | INPUT_SLOT.
- return INPUT_PASS_TO_HANDLERS.

REQ-8: input_defuzz_abs_event(value, old, fuzz):
- if fuzz == 0: return value.
- |value-old| < fuzz/2: return old (within deadband — no change).
- |value-old| < fuzz: return (3·old + value) / 4 (heavy smoothing).
- |value-old| < 2·fuzz: return (old + value) / 2 (light smoothing).
- else: return value.

REQ-9: input_event_dispose:
- if (disposition & PASS_TO_DEVICE) ∧ dev.event: dev.event(dev, type, code, value).
- if disposition & PASS_TO_HANDLERS:
  - if INPUT_SLOT: append EV_ABS/ABS_MT_SLOT triplet first.
  - append triplet to dev.vals[num_vals++].
- if INPUT_FLUSH:
  - if num_vals ≥ 2: input_pass_values(dev, vals, num_vals).
  - num_vals = 0; timestamp[INPUT_CLK_MONO] = 0.
- else if num_vals ≥ max_vals - 2: append SYN_REPORT autoflush + pass + reset.

REQ-10: input_pass_values (RCU walk under dev.event_lock):
- if grab = rcu_dereference(dev.grab): count = grab.handle_events(grab, vals, count); skip list-walk.
- else: for each handle in dev.h_list (RCU): if handle.open → count = handle.handle_events(handle, vals, count); break if count == 0.
- /* Autorepeat hook */ if EV_REP ∧ EV_KEY set: for each v: if v.type == EV_KEY ∧ v.value != 2: start_autorepeat(code) if value, else stop_autorepeat().

REQ-11: input_start_autorepeat:
- if EV_REP set ∧ rep[REP_PERIOD] ∧ rep[REP_DELAY] ∧ timer.function:
  - dev.repeat_key = code.
  - mod_timer(&dev.timer, jiffies + msecs_to_jiffies(rep[REP_DELAY])).

REQ-12: input_repeat_key (timer cb):
- guard(spinlock_irqsave)(&dev.event_lock).
- if !inhibited ∧ test_bit(repeat_key, key) ∧ is_event_supported(repeat_key, keybit, KEY_MAX):
  - input_set_timestamp(dev, ktime_get()).
  - input_handle_event(dev, EV_KEY, repeat_key, 2).
  - input_handle_event(dev, EV_SYN, SYN_REPORT, 1).
  - if rep[REP_PERIOD]: mod_timer(&dev.timer, jiffies + msecs_to_jiffies(rep[REP_PERIOD])).

REQ-13: input_inject_event(handle, type, code, value):
- if !is_event_supported(type, dev.evbit, EV_MAX): return.
- guard(spinlock_irqsave)(&dev.event_lock); guard(rcu)().
- grab = rcu_dereference(dev.grab).
- if !grab ∨ grab == handle: input_handle_event(dev, type, code, value).
- else: drop (foreign-handle while grabbed).

REQ-14: input_grab_device:
- guard(mutex_intr)(&dev.mutex) — -EINTR on signal.
- if dev.grab: return -EBUSY.
- rcu_assign_pointer(dev.grab, handle).
- return 0.

REQ-15: input_release_device / __input_release_device:
- under dev.mutex.
- if rcu_dereference_protected(dev.grab) == handle:
  - rcu_assign_pointer(dev.grab, NULL).
  - synchronize_rcu() — guarantees input_pass_values sees NULL grab.
  - for each open handle in dev.h_list: if handler.start: handler.start(handle) — resync.

REQ-16: input_open_device:
- guard(mutex_intr)(&dev.mutex).
- if dev.going_away: return -ENODEV.
- handle.open++.
- if handler.passive_observer: return 0 (skip users++).
- if dev.users++ ∨ dev.inhibited: return 0 (already open / inhibited).
- if dev.open: error = dev.open(dev); if error: rollback users-- and open--; synchronize_rcu; return error.
- if dev.poller: input_dev_poller_start(dev.poller).

REQ-17: input_close_device:
- guard(mutex)(&dev.mutex).
- __input_release_device(handle) — drops grab if held.
- if !passive_observer ∧ --dev.users == 0 ∧ !dev.inhibited:
  - if poller: stop.
  - if dev.close: dev.close(dev).
- if --handle.open == 0: synchronize_rcu().

REQ-18: input_flush_device:
- guard(mutex_intr).
- if dev.flush: return dev.flush(dev, file).
- else: return 0.

REQ-19: input_register_device:
- if EV_ABS set ∧ !absinfo: return -EINVAL.
- if devres_managed: alloc devres node for unregister.
- __set_bit(EV_SYN, evbit); __clear_bit(KEY_RESERVED, keybit).
- input_cleanse_bitmasks(dev) — zero unannounced sub-bitmaps.
- input_device_tune_vals(dev) — resize vals[] to ≥ estimate + 2.
- if !rep[REP_DELAY] ∧ !rep[REP_PERIOD]: input_enable_softrepeat(dev, 250, 33).
- if !getkeycode: getkeycode = default; if !setkeycode: setkeycode = default.
- if poller: input_dev_poller_finalize.
- device_add(&dev.dev).
- guard(mutex_intr)(&input_mutex):
  - list_add_tail(&dev.node, &input_dev_list).
  - for each handler in input_handler_list: input_attach_handler(dev, handler).
  - input_wakeup_procfs_readers().
- if devres_managed: devres_add(parent, devres).

REQ-20: input_unregister_device / __input_unregister_device:
- input_disconnect_device(dev) — going_away=true; release_keys + SYN_REPORT; clear handle.open.
- guard(mutex)(&input_mutex):
  - for each handle in dev.h_list: handler.disconnect(handle).
  - WARN_ON(!list_empty(&dev.h_list)).
  - timer_delete_sync(&dev.timer).
  - list_del_init(&dev.node).
  - input_wakeup_procfs_readers().
- device_del(&dev.dev).
- if devres_managed: WARN_ON(devres_destroy(... unregister)); input_put_device deferred to second devres.
- else: input_put_device(dev).

REQ-21: input_register_handler:
- input_handler_check_methods — at most one of {filter, event, events}.
- guard(mutex_intr)(&input_mutex):
  - INIT_LIST_HEAD(&handler.h_list).
  - list_add_tail(&handler.node, &input_handler_list).
  - for each dev in input_dev_list: input_attach_handler(dev, handler).
  - input_wakeup_procfs_readers().

REQ-22: input_unregister_handler:
- guard(mutex)(&input_mutex).
- for each handle in handler.h_list: handler.disconnect(handle).
- WARN_ON(!list_empty(&handler.h_list)).
- list_del_init(&handler.node).

REQ-23: input_match_device / input_match_device_id:
- For each id in handler.id_table (until id.flags == 0):
  - if MATCH_BUS / VENDOR / PRODUCT / VERSION flag set: compare; mismatch → skip.
  - bitmap_subset checks on id.{evbit, keybit, relbit, absbit, mscbit, ledbit, sndbit, ffbit, swbit, propbit} ⊆ dev.{respective}.
  - if handler.match: also call handler.match(handler, dev).
- Return matching id or NULL.

REQ-24: input_attach_handler:
- id = input_match_device(handler, dev).
- if !id: return -ENODEV.
- error = handler.connect(handler, dev, id).
- if error ∧ error != -ENODEV: pr_err and propagate.

REQ-25: input_register_handle:
- input_handle_setup_event_handler(handle) — select handle_events impl.
- guard(mutex_intr)(&dev.mutex):
  - if handler.filter: list_add_rcu(d_node, &dev.h_list) (head).
  - else: list_add_tail_rcu(d_node, &dev.h_list) (tail).
- list_add_tail_rcu(h_node, &handler.h_list) (no extra lock — serialized by connect/disconnect mutual exclusion).
- if handler.start: handler.start(handle).

REQ-26: input_unregister_handle:
- list_del_rcu(&handle.h_node).
- guard(mutex)(&dev.mutex): list_del_rcu(&handle.d_node).
- synchronize_rcu() — drain any in-flight input_pass_values.

REQ-27: input_handle_setup_event_handler:
- if handler.filter: handle_events = input_handle_events_filter — applies filter, compacts vals[].
- else if handler.event: handle_events = input_handle_events_default — per-event call to handler.event.
- else if handler.events: handle_events = handler.events.
- else: handle_events = input_handle_events_null — pass-through.

REQ-28: input_allocate_device:
- atomic counter "input_no" produces "inputN" name.
- max_vals = 10 initial (SYN + 7 KEY/MSC + 2 spare).
- vals = kzalloc(max_vals * sizeof(input_value)).
- mutex_init(&mutex); spin_lock_init(&event_lock).
- timer_setup(&timer, NULL, 0).
- INIT_LIST_HEAD(h_list, node).
- dev.type = &input_dev_type; dev.class = &input_class.
- device_initialize(&dev).
- dev_set_name(&dev, "input%lu", input_no).
- __module_get(THIS_MODULE).

REQ-29: devm_input_allocate_device:
- devres_alloc(devm_input_device_release).
- input_allocate_device.
- input.dev.parent = dev; input.devres_managed = true.
- devres_add.

REQ-30: input_free_device:
- if devres_managed: WARN_ON(devres_destroy(... release)).
- input_put_device(dev) — refcount drop triggers input_dev_release: input_ff_destroy, input_mt_destroy_slots, kfree poller/absinfo/vals/dev; module_put.

REQ-31: input_set_capability(dev, type, code):
- if type < EV_CNT ∧ input_max_code[type] ∧ code > input_max_code[type]: pr_err + dump_stack; return.
- per-type: __set_bit on relevant cap bitmap (keybit / relbit / absbit / mscbit / swbit / ledbit / sndbit / ffbit) — EV_ABS additionally calls input_alloc_absinfo; EV_PWR: no bitmap; default: pr_err + dump_stack.
- __set_bit(type, evbit).

REQ-32: input_set_abs_params(dev, axis, min, max, fuzz, flat):
- __set_bit(EV_ABS, evbit); __set_bit(axis, absbit).
- input_alloc_absinfo (if !absinfo).
- absinfo[axis] = { minimum=min, maximum=max, fuzz=fuzz, flat=flat }.

REQ-33: input_copy_abs(dst, dst_axis, src, src_axis):
- WARN_ON if src lacks EV_ABS or src_axis.
- if !src.absinfo: return.
- input_set_capability(dst, EV_ABS, dst_axis).
- if !dst.absinfo: return.
- dst.absinfo[dst_axis] = src.absinfo[src_axis].

REQ-34: input_set_keycode:
- guard(spinlock_irqsave)(&dev.event_lock).
- error = dev.setkeycode(dev, ke, &old_keycode) — default or driver-specific.
- __clear_bit(KEY_RESERVED, keybit).
- if old_keycode ≤ KEY_MAX ∧ no longer supported ∧ was set in dev.key:
  - input_event_dispose(dev, PASS_TO_HANDLERS, EV_KEY, old_keycode, 0).
  - input_event_dispose(dev, PASS_TO_HANDLERS | FLUSH, EV_SYN, SYN_REPORT, 1) — synthetic key-up.

REQ-35: input_default_setkeycode / _getkeycode:
- keycodesize ∈ {1,2,4} discriminates u8/u16/u32 keycode storage.
- INPUT_KEYMAP_BY_INDEX: ke.index = direct.
- else: input_scancode_to_scalar(ke, &index) — len ∈ {1,2,4}; -EINVAL otherwise.
- if index ≥ keycodemax: -EINVAL.
- setkeycode: old = keycode[index]; keycode[index] = ke.keycode.
- Recompute keybit for old keycode (clear unless still mapped elsewhere); __set_bit(ke.keycode, keybit).

REQ-36: input_reset_device:
- guard(mutex)(&dev.mutex); guard(spinlock_irqsave)(&dev.event_lock).
- input_dev_toggle(dev, true) — restore LEDs / SND / REP rate via dev.event.
- if input_dev_release_keys: input_handle_event(EV_SYN, SYN_REPORT, 1) — synthetic key-up flush.

REQ-37: input_inhibit_device:
- guard(mutex).
- if already inhibited: return 0.
- if dev.users: close + poller_stop.
- guard(spinlock_irq)(&dev.event_lock):
  - input_mt_release_slots.
  - input_dev_release_keys + SYN_REPORT.
  - input_dev_toggle(false) — LEDs/SND off.
- inhibited = true.

REQ-38: input_uninhibit_device:
- guard(mutex).
- if !inhibited: return 0.
- if dev.users: open + poller_start.
- inhibited = false.
- guard(spinlock_irq): input_dev_toggle(true).

REQ-39: input_dev_pm_ops:
- .suspend: release_keys + SYN_REPORT; dev_toggle(false).
- .resume: dev_toggle(true).
- .freeze: release_keys + SYN_REPORT.
- .poweroff: dev_toggle(false).
- .restore: == resume.

REQ-40: input_set_timestamp / _get_timestamp:
- set: timestamp[MONO] = ts; [REAL] = ktime_mono_to_real(ts); [BOOT] = ktime_mono_to_any(ts, TK_OFFS_BOOT).
- get: if MONO == 0: input_set_timestamp(dev, ktime_get()); return &timestamp[0].

REQ-41: input_estimate_events_per_packet:
- mt_slots: from mt.num_slots, or ABS_MT_TRACKING_ID range, or 2 if ABS_MT_POSITION_X, else 0.
- events = mt_slots + 1 (SYN_MT_REPORT + SYN_REPORT).
- EV_ABS: per absbit-set add (mt_axis ? mt_slots : 1).
- EV_REL: add bitmap_weight(relbit).
- +7 for KEY / MSC headroom.

REQ-42: input_device_tune_vals:
- packet = input_estimate_events_per_packet.
- hint_events_per_packet = max(hint, packet).
- max_vals_target = hint + 2.
- if dev.max_vals ≥ target: return 0.
- kcalloc target vals; under event_lock: swap; kfree old.

REQ-43: input_cleanse_bitmasks:
- For each {KEY,REL,ABS,MSC,LED,SND,FF,SW}: if !test_bit(EV_##type, evbit): memset(dev.##bit, 0).

REQ-44: input_disconnect_device:
- guard(mutex): going_away = true.
- guard(spinlock_irq)(&dev.event_lock):
  - if input_dev_release_keys: input_handle_event(EV_SYN, SYN_REPORT, 1).
  - for each handle in dev.h_list: handle.open = 0.

REQ-45: input_dev_release_keys:
- if EV_KEY set: for each set bit in dev.key: input_handle_event(EV_KEY, code, 0); need_sync = true.

REQ-46: input class + sysfs:
- input_class.name = "input"; devnode → "input/<name>".
- input_dev_type.groups = input_dev_attr_groups = { input_dev_attr_group (name, phys, uniq, modalias, properties, inhibited), input_dev_id_attr_group (id/{bustype,vendor,product,version}), input_dev_caps_attr_group (capabilities/{ev,key,rel,abs,msc,led,snd,ff,sw}), input_poller_attribute_group }.
- input_dev_type.release = input_dev_release; .uevent = input_dev_uevent; .pm = input_dev_pm_ops.

REQ-47: input_dev_uevent — emits:
- PRODUCT=<bus/vendor/product/version>.
- NAME, PHYS, UNIQ (if set).
- PROP, EV, and per-type bitmap envs (KEY, REL, ABS, MSC, LED, SND, FF, SW) conditional on evbit.
- MODALIAS="input:bXXXXvXXXXpXXXXeXXXX-e..k..r..a..m..l..s..f..w.." with key-trim "+," when buffer pressure.

REQ-48: /proc/bus/input/{devices,handlers}:
- devices: per-dev block I:/N:/P:/S:/U:/H:/B: lines (bitmap hex).
- handlers: "N: Number=N Name=name [(filter)] [Minor=M]" per registered handler.
- input_devices_poll_wait + input_devices_state monotonic counter feeds EPOLLIN on change.

REQ-49: Minor-number allocation:
- INPUT_MAJOR = 13; INPUT_MAX_CHAR_DEVICES = 1024; INPUT_FIRST_DYNAMIC_DEV = 256.
- input_get_new_minor(legacy_base, legacy_num, allow_dynamic):
  - try ida_alloc_range(legacy_base, legacy_base+num-1).
  - on fail (or no legacy): if allow_dynamic: ida_alloc_range(256, 1023).
- input_free_minor: ida_free.

REQ-50: input_init (subsys_initcall):
- class_register(&input_class).
- input_proc_init — proc/bus/input dir + devices + handlers entries.
- register_chrdev_region(MKDEV(INPUT_MAJOR, 0), 1024, "input").

REQ-51: input_max_code[EV_CNT] table — input_set_capability validates code ≤ max for type.

REQ-52: EV-bit set order: handlers list filters first (head), regular handlers second (tail) — filter chain runs to completion before any non-filter sees the events.

## Acceptance Criteria

- [ ] AC-1: input_event called from IRQ: state-bit update + buffered into vals[]; flushed at SYN_REPORT.
- [ ] AC-2: Duplicate EV_KEY events at same state are dropped (no stuck keys); value==2 (repeat) always passes.
- [ ] AC-3: EV_ABS fuzz: |delta| < fuzz/2 → suppressed; |delta| < fuzz → 3/4 smoothing; |delta| < 2·fuzz → 1/2 smoothing.
- [ ] AC-4: ABS_MT_SLOT staged then flushed on next MT axis when slot changes.
- [ ] AC-5: input_grab_device blocks subsequent grabs (-EBUSY); only grab handle receives events; foreign injects ignored.
- [ ] AC-6: input_release_device + synchronize_rcu: handlers see grab removal before next pass_values; handler.start called on resync.
- [ ] AC-7: input_register_device sets EV_SYN, clears KEY_RESERVED, cleanses unannounced bitmaps, resizes vals[], attaches matching handlers under input_mutex.
- [ ] AC-8: input_unregister_device synthesizes key-up + SYN_REPORT for held keys, disconnects all handles, deletes from input_dev_list.
- [ ] AC-9: input_register_handler with two of {filter, event, events} non-NULL → -EINVAL.
- [ ] AC-10: Filter handles register at h_list head; non-filter at tail; pass_values walks in this order.
- [ ] AC-11: input_repeat_key fires at REP_DELAY then every REP_PERIOD ms while key still pressed and !inhibited.
- [ ] AC-12: PM suspend: pressed keys released + SYN_REPORT; LEDs/SND off. Resume: LEDs/SND restored.
- [ ] AC-13: inhibited=true: closes device, releases keys, masks subsequent events at input_get_disposition.
- [ ] AC-14: input_set_keycode: removing keycode from map for a currently-pressed key emits synthetic key-up + SYN_REPORT.
- [ ] AC-15: input_set_capability on type > EV_CNT or code > input_max_code[type]: pr_err + dump_stack, no bit set.
- [ ] AC-16: input_match_device_id: subset check failure on any cap bitmap rejects the id; explicit MATCH_BUS/VENDOR/PRODUCT/VERSION flags compared.
- [ ] AC-17: /proc/bus/input/devices poll wakes when device list changes.

## Architecture

```
struct InputDev {
  name: Option<&str>,
  phys: Option<&str>,
  uniq: Option<&str>,
  id: InputId { bustype: u16, vendor: u16, product: u16, version: u16 },
  propbit: Bitmap<INPUT_PROP_MAX>,
  evbit: Bitmap<EV_MAX>,
  keybit: Bitmap<KEY_MAX>,
  relbit: Bitmap<REL_MAX>,
  absbit: Bitmap<ABS_MAX>,
  mscbit: Bitmap<MSC_MAX>,
  swbit: Bitmap<SW_MAX>,
  ledbit: Bitmap<LED_MAX>,
  sndbit: Bitmap<SND_MAX>,
  ffbit: Bitmap<FF_MAX>,
  key: Bitmap<KEY_MAX>,     // current state
  led: Bitmap<LED_MAX>,
  snd: Bitmap<SND_MAX>,
  sw:  Bitmap<SW_MAX>,
  absinfo: Option<Box<[InputAbsinfo; ABS_CNT]>>,
  mt: Option<Box<InputMt>>,
  rep: [u32; REP_CNT],
  vals: Box<[InputValue]>,
  num_vals: u32,
  max_vals: u32,
  hint_events_per_packet: u32,
  timer: TimerList,
  repeat_key: u32,
  timestamp: [ktime_t; INPUT_CLK_MAX],
  h_list: List<InputHandle>,   // filters head, regular tail (RCU)
  node: ListNode,              // input_dev_list
  dev: Device,
  mutex: Mutex,                // open/close/grab serialization
  event_lock: SpinlockIrq,     // state-bits + vals[]
  grab: RcuPointer<InputHandle>,
  going_away: bool,
  inhibited: bool,
  users: u32,
  devres_managed: bool,
  open: Option<fn(&mut InputDev) -> i32>,
  close: Option<fn(&mut InputDev)>,
  flush: Option<fn(&mut InputDev, &mut File) -> i32>,
  event: Option<fn(&mut InputDev, type_: u32, code: u32, value: i32) -> i32>,
  getkeycode: fn(&InputDev, &mut InputKeymapEntry) -> i32,
  setkeycode: fn(&mut InputDev, &InputKeymapEntry, &mut u32) -> i32,
  poller: Option<Box<InputDevPoller>>,
}

struct InputHandler {
  name: &str,
  filter: Option<fn(&InputHandle, u32, u32, i32) -> bool>,
  event:  Option<fn(&InputHandle, u32, u32, i32)>,
  events: Option<fn(&InputHandle, &mut [InputValue], u32) -> u32>,
  connect:    fn(&mut InputHandler, &mut InputDev, &InputDeviceId) -> i32,
  disconnect: fn(&mut InputHandle),
  start: Option<fn(&InputHandle)>,
  match_: Option<fn(&InputHandler, &InputDev) -> bool>,
  legacy_minors: bool,
  legacy_base: i32, minor: i32,
  passive_observer: bool,
  id_table: &'static [InputDeviceId],
  h_list: List<InputHandle>,
  node: ListNode,
}

struct InputHandle {
  private: *mut c_void,
  name: &str,
  open: u32,
  dev: *mut InputDev,
  handler: *mut InputHandler,
  d_node: ListNode,   // dev.h_list (RCU)
  h_node: ListNode,   // handler.h_list (RCU)
  handle_events: fn(&InputHandle, &mut [InputValue], u32) -> u32,
}

struct InputValue { type_: u16, code: u16, value: i32 }
```

`Input::event(dev, type, code, value)`:
1. /* Capability gate */
2. if !test_bit(type, dev.evbit, EV_MAX): return.
3. guard(spinlock_irqsave)(&dev.event_lock).
4. Input::handle_event(dev, type, code, value).

`Input::handle_event(dev, type, code, value)`:
1. lockdep_assert_held(&dev.event_lock).
2. disp = Input::get_disposition(dev, type, code, &mut value).
3. if disp == INPUT_IGNORE_EVENT: return.
4. if type != EV_SYN: add_input_randomness(type, code, value).
5. Input::event_dispose(dev, disp, type, code, value).

`Input::get_disposition(dev, type, code, pval) -> u32`:
1. if dev.inhibited: return INPUT_IGNORE_EVENT.
2. match type:
   - EV_SYN: SYN_CONFIG → PASS_TO_ALL; SYN_REPORT → PASS_TO_HANDLERS | FLUSH; SYN_MT_REPORT → PASS_TO_HANDLERS.
   - EV_KEY: only if code in keybit; value==2 → PASS_TO_HANDLERS; else if state-bit differs from value: __change_bit(code, dev.key); PASS_TO_HANDLERS.
   - EV_SW: only if code in swbit ∧ state differs: __change_bit; PASS_TO_HANDLERS.
   - EV_ABS: if code in absbit: return Input::handle_abs_event(dev, code, pval).
   - EV_REL: if code in relbit ∧ value != 0: PASS_TO_HANDLERS.
   - EV_MSC: if code in mscbit: PASS_TO_ALL.
   - EV_LED: if code in ledbit ∧ state differs: __change_bit(code, dev.led); PASS_TO_ALL.
   - EV_SND: if code in sndbit: if state differs __change_bit; PASS_TO_ALL.
   - EV_REP: if code ≤ REP_MAX ∧ value ≥ 0 ∧ rep[code] differs: rep[code] = value; PASS_TO_ALL.
   - EV_FF: if value ≥ 0: PASS_TO_ALL.
   - EV_PWR: PASS_TO_ALL.
3. *pval = value (post-defuzz for ABS).

`Input::handle_abs_event(dev, code, pval) -> u32`:
1. if code == ABS_MT_SLOT: if mt ∧ *pval ∈ [0, mt.num_slots): mt.slot = *pval; return INPUT_IGNORE_EVENT.
2. is_mt = input_is_mt_value(code).
3. if !is_mt: pold = &dev.absinfo[code].value.
4. else if mt: pold = &mt.slots[mt.slot].abs[code - ABS_MT_FIRST]; is_new_slot = mt.slot != dev.absinfo[ABS_MT_SLOT].value.
5. else: pold = None (bypass).
6. if pold:
   - *pval = Input::defuzz_abs_event(*pval, *pold, dev.absinfo[code].fuzz).
   - if *pold == *pval: return INPUT_IGNORE_EVENT.
   - *pold = *pval.
7. if is_new_slot: dev.absinfo[ABS_MT_SLOT].value = mt.slot; return PASS_TO_HANDLERS | INPUT_SLOT.
8. return PASS_TO_HANDLERS.

`Input::event_dispose(dev, disp, type, code, value)`:
1. if (disp & PASS_TO_DEVICE) ∧ dev.event: dev.event(dev, type, code, value).
2. if disp & PASS_TO_HANDLERS:
   - if disp & INPUT_SLOT: dev.vals[dev.num_vals++] = { EV_ABS, ABS_MT_SLOT, dev.mt.slot }.
   - dev.vals[dev.num_vals++] = { type, code, value }.
3. if disp & INPUT_FLUSH:
   - if dev.num_vals ≥ 2: Input::pass_values(dev, &dev.vals, dev.num_vals).
   - dev.num_vals = 0; dev.timestamp[INPUT_CLK_MONO] = 0.
4. else if dev.num_vals ≥ dev.max_vals - 2: append synthetic SYN_REPORT; pass; reset.

`Input::pass_values(dev, vals, count)`:
1. lockdep_assert_held(&dev.event_lock).
2. guard(rcu):
   - if grab = rcu_dereference(dev.grab): count = grab.handle_events(grab, vals, count); break.
   - for handle in rcu_list_iter(&dev.h_list, d_node):
     - if handle.open: count = handle.handle_events(handle, vals, count); if count == 0: break.
3. /* Autorepeat trigger */
4. if test_bit(EV_REP, dev.evbit) ∧ test_bit(EV_KEY, dev.evbit):
   - for v in vals[..count]: if v.type_ == EV_KEY ∧ v.value != 2:
     - if v.value: Input::start_autorepeat(dev, v.code).
     - else: Input::stop_autorepeat(dev).

`Input::inject_event(handle, type, code, value)`:
1. dev = handle.dev.
2. if !test_bit(type, dev.evbit, EV_MAX): return.
3. guard(spinlock_irqsave)(&dev.event_lock); guard(rcu):
   - grab = rcu_dereference(dev.grab).
   - if grab.is_none() ∨ grab == handle: Input::handle_event(dev, type, code, value).

`Input::repeat_key(t)`:
1. dev = container_of(t, InputDev, timer).
2. guard(spinlock_irqsave)(&dev.event_lock).
3. if !dev.inhibited ∧ test_bit(dev.repeat_key, &dev.key) ∧ is_event_supported(dev.repeat_key, dev.keybit, KEY_MAX):
   - Input::set_timestamp(dev, ktime_get()).
   - Input::handle_event(dev, EV_KEY, dev.repeat_key, 2).
   - Input::handle_event(dev, EV_SYN, SYN_REPORT, 1).
   - if dev.rep[REP_PERIOD]: mod_timer(&dev.timer, jiffies + msecs_to_jiffies(dev.rep[REP_PERIOD])).

`Input::register_device(dev) -> Result`:
1. if EV_ABS set ∧ dev.absinfo.is_none(): return Err(-EINVAL).
2. if dev.devres_managed: alloc devres-unregister node.
3. __set_bit(EV_SYN, dev.evbit); __clear_bit(KEY_RESERVED, dev.keybit).
4. Input::cleanse_bitmasks(dev).
5. Input::device_tune_vals(dev)?.
6. if dev.rep == [0,0]: Input::enable_softrepeat(dev, 250, 33).
7. if !dev.getkeycode: dev.getkeycode = Input::default_getkeycode.
8. if !dev.setkeycode: dev.setkeycode = Input::default_setkeycode.
9. if dev.poller: input_dev_poller_finalize(dev.poller).
10. device_add(&dev.dev)?.
11. guard(mutex_intr)(&input_mutex):
    - list_add_tail(&dev.node, &input_dev_list).
    - for handler in input_handler_list: Input::attach_handler(dev, handler).
    - input_wakeup_procfs_readers().
12. if dev.devres_managed: devres_add(parent, devres).

`Input::register_handle(handle) -> Result`:
1. Input::setup_event_handler(handle).
2. guard(mutex_intr)(&handle.dev.mutex):
   - if handle.handler.filter: list_add_rcu(d_node head).
   - else: list_add_tail_rcu(d_node tail).
3. list_add_tail_rcu(&handle.h_node, &handle.handler.h_list).
4. if handle.handler.start: handle.handler.start(handle).

`Input::setup_event_handler(handle)`:
1. h = handle.handler.
2. if h.filter: handle.handle_events = Input::handle_events_filter.
3. else if h.event: handle.handle_events = Input::handle_events_default.
4. else if h.events: handle.handle_events = h.events.
5. else: handle.handle_events = Input::handle_events_null.

`Input::handle_events_filter(handle, vals, count)`:
1. /* In-place compact retaining un-filtered events */
2. end = 0.
3. for v in vals[..count]:
   - if !handle.handler.filter(handle, v.type_, v.code, v.value):
     - if end != current_index: vals[end] = v.
     - end += 1.
4. return end.

`Input::start_autorepeat(dev, code)`:
1. if test_bit(EV_REP, dev.evbit) ∧ dev.rep[REP_PERIOD] != 0 ∧ dev.rep[REP_DELAY] != 0 ∧ dev.timer.function.is_some():
   - dev.repeat_key = code.
   - mod_timer(&dev.timer, jiffies + msecs_to_jiffies(dev.rep[REP_DELAY])).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `event_lock_held_in_dispose` | INVARIANT | per-event_dispose: event_lock held; lockdep assert. |
| `vals_capacity_never_exceeded` | INVARIANT | per-event_dispose: num_vals ≤ max_vals - 1 before append; autoflush at max_vals - 2. |
| `grab_singleton` | INVARIANT | per-grab_device: only one handle may be grab at any time (-EBUSY otherwise). |
| `grab_synchronize_rcu_on_release` | INVARIANT | per-release_device: synchronize_rcu before handler.start to ensure pass_values sees NULL grab. |
| `filter_handler_head_handler_tail` | INVARIANT | per-register_handle: filter → list_add_rcu head; non-filter → list_add_tail_rcu tail. |
| `ev_syn_always_set_on_register` | INVARIANT | per-register_device: EV_SYN forced. |
| `keymap_set_simulates_keyup` | INVARIANT | per-set_keycode: if removing currently-held keycode: synthetic EV_KEY=0 + SYN_REPORT. |
| `inhibited_short_circuits` | INVARIANT | per-get_disposition: inhibited ⟹ INPUT_IGNORE_EVENT regardless of type. |
| `passive_observer_no_users_increment` | INVARIANT | per-open_device: passive_observer ⟹ dev.users unchanged. |
| `repeat_timer_not_fired_when_inhibited` | INVARIANT | per-repeat_key: inhibited ⟹ no event. |
| `abs_defuzz_idempotent_within_deadband` | INVARIANT | per-defuzz_abs_event: |delta| < fuzz/2 ⟹ result == old. |
| `handler_check_methods_exclusive` | INVARIANT | per-register_handler: at most one of filter/event/events. |

### Layer 2: TLA+

`drivers/input/input-core.tla`:
- Per-event-flow + per-grab + per-register/unregister-dev + per-register/unregister-handler + per-autorepeat.
- Properties:
  - `safety_no_event_after_unregister` — per-disconnect_device: no event leaks after handle.open=0.
  - `safety_grab_exclusive` — at most one handle holds dev.grab.
  - `safety_filter_runs_before_handler` — per-pass_values: filter handles consume/pass before regular handles.
  - `safety_state_bit_matches_last_event` — per-EV_KEY/SW/LED: dev.key/sw/led bit reflects last delivered value.
  - `safety_vals_not_overflow` — per-event_dispose: num_vals never > max_vals.
  - `safety_grab_release_drains_in_flight` — per-release_device: synchronize_rcu before any subsequent pass.
  - `liveness_event_eventually_delivered` — per-event(open handle, no grab elsewhere): handle_events called at some flush.
  - `liveness_autorepeat_periodic` — per-key-held + EV_REP set + !inhibited: repeat_key timer re-arms every REP_PERIOD.
  - `liveness_unregister_drains_handles` — per-unregister_device: all handles disconnected before list_del.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Input::event` post: state-bit toggled iff PASS_TO_HANDLERS returned for state-tracked types | `Input::event` |
| `Input::get_disposition` post: returned disposition ∈ {IGNORE, PASS_TO_HANDLERS, PASS_TO_DEVICE, PASS_TO_ALL, +SLOT, +FLUSH} | `Input::get_disposition` |
| `Input::defuzz_abs_event` post: |result-old| ≤ |value-old| | `Input::defuzz_abs_event` |
| `Input::handle_abs_event` post: ABS_MT_SLOT staged not emitted; subsequent MT axis flushes slot | `Input::handle_abs_event` |
| `Input::pass_values` post: count returned ≤ count in; if grab set, only grab.handle_events invoked | `Input::pass_values` |
| `Input::register_device` post: dev in input_dev_list; EV_SYN set; max_vals ≥ estimate + 2 | `Input::register_device` |
| `Input::unregister_device` post: dev removed from list; h_list empty; timer disarmed; pressed keys synthesized off | `Input::unregister_device` |
| `Input::register_handle` post: filter at head, non-filter at tail; handle in handler.h_list; handle_events set | `Input::register_handle` |
| `Input::unregister_handle` post: synchronize_rcu observed; handle removed from both lists | `Input::unregister_handle` |
| `Input::grab_device` post: dev.grab == handle; existing grab ⟹ -EBUSY | `Input::grab_device` |
| `Input::release_device` post: grab cleared; synchronize_rcu; handler.start called for each open handle | `Input::release_device` |
| `Input::set_keycode` post: dev.keybit updated; if displaced live key: synthetic key-up + SYN_REPORT | `Input::set_keycode` |
| `Input::repeat_key` post: under event_lock; emits EV_KEY=2 + SYN_REPORT; re-arms timer at REP_PERIOD | `Input::repeat_key` |
| `Input::set_capability` post: code > input_max_code[type] ⟹ no bit set; pr_err logged | `Input::set_capability` |

### Layer 4: Verus/Creusot functional

`Per-driver report → input_event → input_handle_event → input_get_disposition → input_event_dispose → SYN_REPORT flush → input_pass_values → handle.handle_events (filter/default/events/null) → consumer (evdev/mousedev/kbd)` semantic equivalence: per-Documentation/input/event-codes.rst + Documentation/input/multi-touch-protocol.rst + Documentation/input/uinput.rst.

## Hardening

(Inherits row-1 features from `drivers/input/00-overview.md` § Hardening.)

Input-core reinforcement:

- **Per-event_lock spinlock_irqsave around all state mutation** — defense against per-IRQ + process-context race on dev.key / vals[].
- **Per-RCU dev.grab read/write** — defense against per-grab-revoke UAF during in-flight pass_values.
- **Per-synchronize_rcu in release_device + unregister_handle + close_device** — defense against per-stale-handle event delivery.
- **Per-handler-method-exclusivity enforced at register** — defense against per-undefined-behavior when filter + event both set.
- **Per-input_mutex protecting input_dev_list + input_handler_list mutual exclusion of register/unregister/attach** — defense against per-mid-attach unregister.
- **Per-EV_ABS device without absinfo refused at register** — defense against per-NULL-absinfo deref in handle_abs_event.
- **Per-going_away flag checked at open_device under mutex** — defense against per-open-after-unregister.
- **Per-KEY_RESERVED never emitted** — defense against per-uapi reserved-code leakage.
- **Per-input_set_capability bounds-check vs input_max_code[type]** — defense against per-bitmap overflow on user-supplied caps.
- **Per-vals[] autoflush at max_vals - 2** — defense against per-buffer overflow during burst.
- **Per-disconnect_device synthesizes key-up + SYN_REPORT** — defense against per-stuck-key after device removal.
- **Per-suspend / freeze release_keys** — defense against per-stuck-key across PM cycle.
- **Per-inhibited short-circuit at get_disposition** — defense against per-event-leak through inhibited device.
- **Per-set_keycode keymap mutation synthesizes key-up for displaced live keys** — defense against per-stuck-key on remap.
- **Per-input_handler_for_each_handle RCU read-only** — defense against per-list-mutation in callback.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/input/evdev.c per-device /dev/input/eventN cdev (covered separately if expanded)
- drivers/input/mousedev.c, joydev.c, keyboard.c handlers (covered separately if expanded)
- drivers/input/input-mt.c multi-touch slot machinery (covered separately if expanded)
- drivers/input/input-poller.c poller worker thread (covered separately if expanded)
- drivers/input/ff-core.c force-feedback (covered separately if expanded)
- drivers/input/input-leds.c LED-trigger bridging (covered separately if expanded)
- drivers/input/input-compat.c 32/64-bit ioctl compat (covered separately if expanded)
- Implementation code
