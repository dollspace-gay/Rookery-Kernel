# Tier-3: drivers/pwm/core.c — Generic PWM subsystem core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pwm/00-overview.md
upstream-paths:
  - drivers/pwm/core.c (~2759 lines)
  - include/linux/pwm.h
  - include/uapi/linux/pwm.h
  - include/dt-bindings/pwm/pwm.h
  - include/trace/events/pwm.h
-->

## Summary

The **PWM (Pulse-Width Modulation) subsystem core** provides a generic framework for chips that emit periodic on/off signals (backlights, fans, motor controllers, IR LEDs, audio piezos). A *PWM chip* (`struct pwm_chip`) hosts `npwm` `struct pwm_device` lines and dispatches consumer requests through `struct pwm_ops`. Drivers register chips via `pwmchip_alloc()` + `__pwmchip_add()` (wrapped by `pwmchip_add()`); consumers look up lines via `pwm_get(dev, con_id)` (DT `pwms` / `pwm-names` ⇒ `of_pwm_get`; ACPI ⇒ `acpi_pwm_get`; fallback lookup table) and apply state via `pwm_apply_might_sleep(pwm, &state)` (sleep-OK) or `pwm_apply_atomic(pwm, &state)` (chip-marked `atomic`). State is `{period, duty_cycle, polarity, enabled, usage_power}` in ns. A newer **waveform API** (`round_waveform_tohw` / `round_waveform_fromhw` / `read_waveform` / `write_waveform` with per-driver opaque `wfhw` buffer of size `PWM_WFHWSIZE` and `sizeof_wfhw`) lets drivers expose `{period_length_ns, duty_length_ns, duty_offset_ns}` directly and is round-trippable through `pwm_round_waveform_might_sleep` / `pwm_set_waveform_might_sleep(exact)` / `pwm_get_waveform_might_sleep`. Sysfs (`/sys/class/pwm/pwmchip<N>/`) and a UAPI character device (`PWM_IOCTL_REQUEST` / `_FREE` / `_ROUNDWF` / `_GETWF` / `_SETROUNDEDWF` / `_SETEXACTWF`) expose chips to userspace; an optional `gpiochip` shim exposes the chip as binary GPIO when the driver implements `write_waveform`. Capture-API (`pwm_capture()` via `ops->capture`) reports observed period/duty on a PWM-capable input. Critical for: laptop backlight, server fan PWM, embedded motor + buzzer + IR-LED control.

This Tier-3 covers `drivers/pwm/core.c` (~2759 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pwm_chip` | per-chip | `PwmChip` |
| `struct pwm_device` | per-line | `PwmDevice` |
| `struct pwm_state` | per-state | `PwmState` |
| `struct pwm_args` | per-bootloader-args | `PwmArgs` |
| `struct pwm_waveform` | per-waveform | `PwmWaveform` |
| `struct pwm_capture` | per-capture-result | `PwmCapture` |
| `struct pwm_ops` | per-driver vtable | `PwmOps` |
| `struct pwm_lookup` | per-board-table-entry | `PwmLookup` |
| `pwmchip_alloc()` / `devm_pwmchip_alloc()` | per-allocate | `Pwm::chip_alloc` / `devm_chip_alloc` |
| `pwmchip_put()` / `pwmchip_release()` | per-free | `Pwm::chip_put` / `chip_release` |
| `__pwmchip_add()` / `__devm_pwmchip_add()` | per-register | `Pwm::chip_add` / `devm_chip_add` |
| `pwmchip_remove()` | per-deregister | `Pwm::chip_remove` |
| `pwm_get()` / `devm_pwm_get()` | per-consumer-lookup | `Pwm::get` / `devm_get` |
| `of_pwm_get()` (static) | per-DT-lookup | `Pwm::of_get` |
| `acpi_pwm_get()` (static) | per-ACPI-lookup | `Pwm::acpi_get` |
| `devm_fwnode_pwm_get()` | per-fwnode-lookup | `Pwm::devm_fwnode_get` |
| `of_pwm_xlate_with_flags()` / `of_pwm_single_xlate()` | per-DT-cell-decoder | `Pwm::of_xlate_with_flags` / `of_single_xlate` |
| `pwm_put()` | per-release | `Pwm::put` |
| `pwm_request_from_chip()` (static) | per-line-request | `Pwm::request_from_chip` |
| `pwm_device_request()` (static) | per-line-acquire | `Pwm::device_request` |
| `pwm_apply_might_sleep()` | per-state-apply (sleep) | `Pwm::apply_might_sleep` |
| `pwm_apply_atomic()` | per-state-apply (atomic) | `Pwm::apply_atomic` |
| `pwm_get_state_hw()` | per-readback | `Pwm::get_state_hw` |
| `pwm_adjust_config()` | per-bootloader-handover | `Pwm::adjust_config` |
| `pwm_round_waveform_might_sleep()` | per-waveform-round | `Pwm::round_waveform` |
| `pwm_set_waveform_might_sleep()` | per-waveform-set | `Pwm::set_waveform` |
| `pwm_get_waveform_might_sleep()` | per-waveform-read | `Pwm::get_waveform` |
| `pwm_capture()` (static) | per-capture | `Pwm::capture` |
| `pwm_add_table()` / `pwm_remove_table()` | per-board-table | `Pwm::add_table` / `remove_table` |
| `pwm_class` / `pwm_class_pm_ops` | per-sysfs-class + PM | `PWM_CLASS` |
| `pwm_cdev_fileops` / `pwm_cdev_ioctl` | per-UAPI cdev | `Pwm::cdev_fileops` / `cdev_ioctl` |
| `PWM_IOCTL_REQUEST` / `_FREE` / `_ROUNDWF` / `_GETWF` / `_SETROUNDEDWF` / `_SETEXACTWF` | per-UAPI ioctl | `PwmIoctl` |
| `PWMF_REQUESTED` / `PWMF_EXPORTED` | per-line-flag | `PwmFlags` |
| `PWM_WFHWSIZE` | per-wfhw-buffer-cap | `PWM_WFHWSIZE` |
| `PWM_MINOR_COUNT` | per-cdev-minor-count | `PWM_MINOR_COUNT` |
| `pwm_chips` (IDR) / `pwm_lock` | per-global-registry | `PWM_CHIPS` |
| `pwm_lookup_list` / `pwm_lookup_lock` | per-board-table | `PWM_LOOKUP` |
| `pwm_wf_valid()` / `pwm_state_valid()` (static) | per-validation | `Pwm::wf_valid` / `state_valid` |
| `pwm_wf2state()` / `pwm_state2wf()` (static) | per-conversion | `Pwm::wf2state` / `state2wf` |
| `pwm_apply_debug()` (static) | per-CONFIG_PWM_DEBUG check | `Pwm::apply_debug` |

## Compatibility contract

REQ-1: struct pwm_chip:
- dev: embedded `struct device` (per-pwm_class).
- cdev: per-UAPI char device.
- ops: `const struct pwm_ops *`.
- owner: module owner (for try_module_get).
- id: idr-allocated chip id (`pwmchip<id>` sysfs name).
- npwm: number of lines.
- pwms: flex array of `struct pwm_device` (length npwm).
- atomic: per-chip flag — `apply` / `write_waveform` must not sleep ⟹ `atomic_lock` (spin) instead of `nonatomic_lock` (mutex).
- atomic_lock / nonatomic_lock: per-chip serialization of `pwm_ops` invocations.
- operational: bool — `true` after `__pwmchip_add` returns success, `false` again in `pwmchip_remove`.
- uses_pwmchip_alloc: stamped by `pwmchip_alloc` (required for `__pwmchip_add` to accept the chip).
- of_xlate: DT-phandle decoder (`of_pwm_xlate_with_flags` default if NULL and DT node present).
- gpio: optional gpio_chip facade when CONFIG_PWM_PROVIDE_GPIO and driver provides `write_waveform`.

REQ-2: struct pwm_device:
- chip: back-pointer to `pwm_chip`.
- hwpwm: per-chip index.
- label: human-readable consumer label (set on request).
- flags: bitmap of `PWMF_REQUESTED` / `PWMF_EXPORTED`.
- args: per-bootloader-args (period, polarity) from DT/ACPI/lookup-table.
- state: per-currently-applied state.
- last: per-last-debug snapshot (CONFIG_PWM_DEBUG only).

REQ-3: struct pwm_state:
- period: u64 ns.
- duty_cycle: u64 ns (≤ period when enabled).
- polarity: `PWM_POLARITY_NORMAL` (0) or `PWM_POLARITY_INVERSED` (1).
- enabled: bool.
- usage_power: bool — driver may choose lowest-power waveform (e.g. for backlight where any duty ≤ value is acceptable).

REQ-4: struct pwm_waveform:
- period_length_ns: u64 (≤ S64_MAX; reserves negative-encoding for future polarity-inverted form).
- duty_length_ns: u64 (≤ period_length_ns).
- duty_offset_ns: u64 (< period_length_ns unless both 0).

REQ-5: struct pwm_ops vtable:
- request(chip, pwm) -> int (optional).
- free(chip, pwm) (optional).
- capture(chip, pwm, result, timeout_ms) -> int (optional; capture API).
- apply(chip, pwm, state) -> int (legacy path; required if write_waveform absent).
- get_state(chip, pwm, state) -> int (optional; recommended for CONFIG_PWM_DEBUG).
- round_waveform_tohw(chip, pwm, wf, wfhw) -> int (waveform path; required if write_waveform present).
- round_waveform_fromhw(chip, pwm, wfhw, wf) -> int (waveform path; required if write_waveform present).
- read_waveform(chip, pwm, wfhw) -> int (waveform path; optional).
- write_waveform(chip, pwm, wfhw) -> int (waveform path; presence selects waveform API).
- sizeof_wfhw: size of opaque hw buffer; must be ≤ `PWM_WFHWSIZE` (= 20 currently).

REQ-6: `pwm_ops_check(chip)` registration gate:
- If ops.write_waveform:
  - require ops.round_waveform_tohw ∧ ops.round_waveform_fromhw.
  - require PWM_WFHWSIZE ≥ ops.sizeof_wfhw (else dev_warn + reject).
- else:
  - require ops.apply.
  - if CONFIG_PWM_DEBUG ∧ !ops.get_state: dev_warn.

REQ-7: pwmchip_alloc(parent, npwm, sizeof_priv):
- alloc_size = ALIGN(struct_size(chip, pwms, npwm), PWMCHIP_ALIGN) + sizeof_priv (overflow-checked via size_add).
- kzalloc(alloc_size, GFP_KERNEL); chip.npwm = npwm; chip.uses_pwmchip_alloc = true; chip.operational = false.
- device_initialize(&chip.dev); chip.dev.class = &pwm_class; chip.dev.parent = parent; chip.dev.release = pwmchip_release.
- pwmchip_set_drvdata(chip, pwmchip_priv(chip)) — driver-private region trailing the pwms array.
- For each i in 0..npwm: chip.pwms[i].chip = chip; chip.pwms[i].hwpwm = i.

REQ-8: __pwmchip_add(chip, owner):
- Validate: chip ≠ NULL ∧ pwmchip_parent(chip) ∧ chip.ops ∧ chip.npwm ≠ 0.
- Require chip.uses_pwmchip_alloc (else -EINVAL).
- Run pwm_ops_check (else -EINVAL).
- chip.owner = owner.
- if chip.atomic: spin_lock_init(&chip.atomic_lock); else mutex_init(&chip.nonatomic_lock).
- guard(mutex)(&pwm_lock):
  - id = idr_alloc(&pwm_chips, chip, 0, 0, GFP_KERNEL); if id < 0 return id; chip.id = id.
  - dev_set_name(&chip.dev, "pwmchip%u", chip.id).
  - CONFIG_OF: of_pwmchip_add(chip) — defaults of_xlate to of_pwm_xlate_with_flags; takes ref on of_node.
  - scoped_guard(pwmchip, chip): chip.operational = true.
  - If chip.ops.write_waveform: if id < PWM_MINOR_COUNT: chip.dev.devt = MKDEV(MAJOR(pwm_devt), id); else dev_warn.
  - cdev_init(&chip.cdev, &pwm_cdev_fileops); chip.cdev.owner = owner.
  - cdev_device_add(&chip.cdev, &chip.dev). On failure: rollback (operational=false, of_pwmchip_remove, idr_remove).
  - CONFIG_PWM_PROVIDE_GPIO ∧ ops.write_waveform: populate chip.gpio (label=parent name, request/free/get_direction/set, can_sleep=true) and gpiochip_add_data. On failure: rollback cdev_device_del.

REQ-9: pwmchip_remove(chip):
- CONFIG_PWM_PROVIDE_GPIO ∧ ops.write_waveform: gpiochip_remove(&chip.gpio).
- pwmchip_sysfs_unexport(chip) — for each pwm with PWMF_EXPORTED: pwm_unexport_child.
- scoped_guard(mutex, &pwm_lock):
  - scoped_guard(pwmchip, chip): chip.operational = false.
  - For each i in 0..npwm: if test_and_clear_bit(PWMF_REQUESTED, &pwm.flags): dev_warn "Freeing requested PWM #i"; if ops.free: ops.free(chip, pwm).
  - CONFIG_OF: of_pwmchip_remove(chip).
  - idr_remove(&pwm_chips, chip.id).
- cdev_device_del(&chip.cdev, &chip.dev).

REQ-10: pwm_apply_might_sleep(pwm, state):
- might_sleep().
- guard(pwmchip)(chip) /* mutex for !atomic, spin for atomic */.
- if !chip.operational: return -ENODEV.
- CONFIG_PWM_DEBUG ∧ chip.atomic: non_block_start()/__pwm_apply()/non_block_end() — sleep-detect trap.
- else: __pwm_apply(pwm, state).

REQ-11: pwm_apply_atomic(pwm, state):
- WARN_ONCE(!chip.atomic, "sleeping PWM driver used in atomic context").
- guard(pwmchip)(chip).
- if !chip.operational: -ENODEV.
- return __pwm_apply(pwm, state).

REQ-12: __pwm_apply(pwm, state):
- Validate pointers; pwm_state_valid(state) ⟹ enabled ⟹ period > 0 ∧ duty_cycle ≤ period. If invalid and current state is also invalid: accept (allows polarity-then-period sysfs flow); else -EINVAL.
- No-op fast path: if (period, duty_cycle, polarity, enabled, usage_power) match current pwm.state: return 0.
- If pwmchip_supports_waveform(chip):
  - pwm_state2wf(state, &wf).
  - __pwm_round_waveform_tohw(chip, pwm, &wf, &wfhw). >0 ⟹ -EINVAL (request exceeds hw limits).
  - CONFIG_PWM_DEBUG: round_fromhw + pwm_check_rounding(wf, wf_rounded) ⟹ dev_err on rounding bug.
  - __pwm_write_waveform(chip, pwm, &wfhw).
  - pwm.state = *state.
- else:
  - ops.apply(chip, pwm, state); trace_pwm_apply.
  - pwm.state = *state.
  - pwm_apply_debug(pwm, state) /* readback + reapply idempotency check */.

REQ-13: pwm_round_waveform_might_sleep(pwm, wf):
- BUG_ON(PWM_WFHWSIZE < ops.sizeof_wfhw).
- pwmchip_supports_waveform ⟹ if !supported: -EOPNOTSUPP.
- pwm_wf_valid(wf) /* period ≤ S64_MAX ∧ duty ≤ period ∧ (offset == 0 ∨ offset < period) */ ⟹ -EINVAL.
- guard(pwmchip)(chip); operational check.
- ret_tohw = __pwm_round_waveform_tohw(chip, pwm, wf, wfhw); negative ⟹ propagate.
- ret_fromhw = __pwm_round_waveform_fromhw(chip, pwm, wfhw, wf); negative ⟹ propagate.
- Return ret_tohw (0 = exact, 1 = at least one value rounded up).

REQ-14: pwm_set_waveform_might_sleep(pwm, wf, exact):
- might_sleep(); guard(pwmchip)(chip); operational check.
- if !chip.atomic ∨ debug-disabled: err = __pwm_set_waveform(pwm, wf, exact).
- else: non_block_start()/__pwm_set_waveform/non_block_end().
- if err == -EDOM: err = -EINVAL (defensive; __pwm_set_waveform should never return -EDOM).
- else if exact ∧ err == 1: err = -EDOM (rounding occurred but caller wanted exact).
- else if err == 1: err = 0 (rounding ack'd in !exact mode).
- return err.

REQ-15: pwm_get_waveform_might_sleep(pwm, wf):
- pwmchip_supports_waveform ∧ ops.read_waveform required else -EOPNOTSUPP.
- guard(pwmchip)(chip); operational check.
- __pwm_read_waveform(chip, pwm, &wfhw); __pwm_round_waveform_fromhw(chip, pwm, &wfhw, wf).

REQ-16: pwm_get_state_hw(pwm, state):
- might_sleep(); guard(pwmchip)(chip); operational check.
- if pwmchip_supports_waveform ∧ ops.read_waveform:
  - __pwm_read_waveform; __pwm_round_waveform_fromhw -> wf; pwm_wf2state(wf, state).
- else if ops.get_state: ops.get_state(chip, pwm, state); trace_pwm_get.
- else: -EOPNOTSUPP.

REQ-17: pwm_get(dev, con_id):
- fwnode = dev_fwnode(dev) (NULL OK).
- If is_of_node(fwnode): return of_pwm_get(dev, to_of_node(fwnode), con_id).
- Else if is_acpi_node(fwnode): try acpi_pwm_get(fwnode); if Ok ∨ err != -ENOENT: return.
- Else fall through to legacy `pwm_lookup_list` (board-supplied table):
  - Most-specific match wins (dev+con > dev-only > con-only); NULL ID is wildcard.
  - Resolve provider via pwmchip_find_by_name; if absent and chosen.module present: request_module + retry; else -EPROBE_DEFER.
  - pwm_request_from_chip(chip, chosen.index, con_id ?: dev_id).
  - pwm_device_link_add(dev, pwm); pwm.args.period = chosen.period; pwm.args.polarity = chosen.polarity.

REQ-18: of_pwm_get(dev, np, con_id):
- If con_id: index = of_property_match_string(np, "pwm-names", con_id); index < 0 ⟹ propagate.
- of_parse_phandle_with_args_map(np, "pwms", "pwm", index, &args).
- chip = fwnode_to_pwmchip(of_fwnode_handle(args.np)); else -EPROBE_DEFER.
- pwm = chip.of_xlate(chip, &args).
- pwm_device_link_add(dev, pwm) — on failure pwm_put(pwm).
- If no con_id supplied: of_property_read_string_index(np, "pwm-names", index, &con_id); fallback np.name.
- pwm.label = con_id.

REQ-19: of_pwm_xlate_with_flags(chip, args):
- args.args_count ≥ 1 required.
- pwm = pwm_request_from_chip(chip, args.args[0], NULL).
- If args.args_count > 1: pwm.args.period = args.args[1].
- pwm.args.polarity = (args.args_count > 2 ∧ args.args[2] & PWM_POLARITY_INVERTED) ? PWM_POLARITY_INVERSED : PWM_POLARITY_NORMAL.

REQ-20: acpi_pwm_get(fwnode):
- __acpi_node_get_property_reference(fwnode, "pwms", 0, 3, &args).
- args.nargs ≥ 2 (chip + index + period [+ flags]); else -EPROTO.
- chip via fwnode_to_pwmchip; pwm_request_from_chip(chip, args.args[0], NULL).
- pwm.args.period = args.args[1]; polarity = (args.nargs > 2 ∧ args.args[2] & PWM_POLARITY_INVERTED) ? INVERSED : NORMAL.

REQ-21: pwm_request_from_chip(chip, index, label):
- chip ≠ NULL ∧ index < chip.npwm.
- guard(mutex)(&pwm_lock).
- pwm = &chip.pwms[index]; err = pwm_device_request(pwm, label).
- pwm_device_request:
  - test_bit(PWMF_REQUESTED, &pwm.flags) ⟹ -EBUSY.
  - !chip.operational ⟹ -ENODEV.
  - try_module_get(chip.owner) ⟹ -ENODEV.
  - get_device(&chip.dev) ⟹ -ENODEV; if ops.request fails: put_device + module_put + propagate.
  - If ops.read_waveform ∨ ops.get_state: pwm_get_state_hw(pwm, &state); if Ok: pwm.state = state; (PWM_DEBUG: pwm.last = pwm.state).
  - set_bit(PWMF_REQUESTED, &pwm.flags); pwm.label = label.

REQ-22: pwm_put(pwm) / __pwm_put(pwm):
- guard(mutex)(&pwm_lock) for pwm_put.
- If chip.operational ∧ !test_and_clear_bit(PWMF_REQUESTED, &pwm.flags): pr_warn "PWM device already freed"; return (double-free guard).
- If chip.operational ∧ ops.free: ops.free(chip, pwm).
- pwm.label = NULL; put_device(&chip.dev); module_put(chip.owner).

REQ-23: pwm_capture(pwm, result, timeout_ms):
- if !ops.capture: -ENOSYS.
- guard(mutex)(&pwm_lock); guard(pwmchip)(chip); operational check.
- ops.capture(chip, pwm, result, timeout_ms).

REQ-24: pwm_adjust_config(pwm):
- pwm_get_args(pwm, &pargs); pwm_get_state(pwm, &state).
- If state.period == 0: state.duty_cycle = 0; state.period = pargs.period; state.polarity = pargs.polarity; pwm_apply_might_sleep.
- Else if pargs.period ≠ state.period: scale state.duty_cycle ← duty_cycle * pargs.period / state.period; state.period = pargs.period.
- If pargs.polarity ≠ state.polarity: state.polarity = pargs.polarity; state.duty_cycle = state.period - state.duty_cycle.
- pwm_apply_might_sleep(pwm, &state).

REQ-25: UAPI character device:
- alloc_chrdev_region(&pwm_devt, 0, PWM_MINOR_COUNT=256, "pwm") at subsys_initcall.
- pwm_cdev_open: cdata = kzalloc(struct pwm_cdev_data + pwm[npwm]); nonseekable_open.
- pwm_cdev_release: for each requested cdata.pwm[i]: pwm_put + free label.
- pwm_cdev_ioctl (under guard(mutex)(&pwm_lock) and !operational ⟹ -ENODEV):
  - PWM_IOCTL_REQUEST(arg=hwpwm): pwm_device_request with "pwm-cdev (pid=<x>)" label.
  - PWM_IOCTL_FREE(arg=hwpwm): __pwm_put.
  - PWM_IOCTL_ROUNDWF: copy_from_user(struct pwmchip_waveform); __pad must be 0; pwm_round_waveform_might_sleep; copy_to_user rounded result.
  - PWM_IOCTL_GETWF: pwm_get_waveform_might_sleep; copy_to_user.
  - PWM_IOCTL_SETROUNDEDWF / _SETEXACTWF: pwm_wf_valid; pwm_set_waveform_might_sleep(pwm, &wf, exact = (_SETEXACTWF)); ret == 1 ⟹ 0.
  - default: -ENOTTY.

REQ-26: pwm sysfs (`/sys/class/pwm/pwmchip<N>/`):
- chip attributes: `export` (WO), `unexport` (WO), `npwm` (RO).
- per-pwm attributes (after export): `period` (RW), `duty_cycle` (RW), `enable` (RW), `polarity` (RW: "normal"/"inversed"), `capture` (RO: triggers pwm_capture with HZ=1s timeout).
- PWM_class PM (DEFINE_SIMPLE_DEV_PM_OPS):
  - suspend: for each exported pwm, snapshot state and disable; on error roll back the ones already touched.
  - resume: restore previously enabled state.

REQ-27: GPIO facade (CONFIG_PWM_PROVIDE_GPIO + write_waveform):
- gpio_chip {.can_sleep = true, .ngpio = npwm, .label = parent dev_name, .request/free/get_direction=OUT/set = pwm_gpio_*}.
- pwm_gpio_set(value): wf = {period_length_ns=1}; pwm_round_waveform_might_sleep; wf.duty_length_ns = value ? wf.period_length_ns : 0; pwm_set_waveform_might_sleep(pwm, &wf, true).

REQ-28: Per-chip locking model:
- chip.atomic ⟹ spin_lock(&chip.atomic_lock); pwm_apply_atomic legal.
- !chip.atomic ⟹ mutex_lock(&chip.nonatomic_lock); pwm_apply_might_sleep only.
- `DEFINE_GUARD(pwmchip, ...)` wraps pwmchip_lock/_unlock.

REQ-29: Tracepoints:
- trace_pwm_apply, trace_pwm_get, trace_pwm_round_waveform_tohw, trace_pwm_round_waveform_fromhw, trace_pwm_read_waveform, trace_pwm_write_waveform — emitted on every driver hook invocation.

REQ-30: Debug fs (`/sys/kernel/debug/pwm`):
- seq_file iterator over `pwm_chips` IDR; per-chip lists requested status + requested config + actual config (via read_waveform or get_state_hw).

## Acceptance Criteria

- [ ] AC-1: pwmchip_alloc(npwm=0): allocates chip but __pwmchip_add later -EINVAL'd. pwmchip_alloc(npwm>0): chip.uses_pwmchip_alloc=true, chip.operational=false, pwms[i].chip set, pwms[i].hwpwm=i.
- [ ] AC-2: __pwmchip_add with !uses_pwmchip_alloc: -EINVAL (catch drivers using stack/manual struct).
- [ ] AC-3: __pwmchip_add with ops.write_waveform but missing round_waveform_tohw/_fromhw: -EINVAL via pwm_ops_check.
- [ ] AC-4: __pwmchip_add with ops.sizeof_wfhw > PWM_WFHWSIZE: dev_warn + -EINVAL.
- [ ] AC-5: pwmchip_add then pwmchip_remove: chip.operational toggles true→false; PWMF_REQUESTED lines warn + free.
- [ ] AC-6: pwm_get(dev) with DT pwms="<phandle> <idx>": resolves to chip via fwnode_to_pwmchip; -EPROBE_DEFER if not registered yet.
- [ ] AC-7: pwm_get with ACPI Package {pwms = {ref, idx, period [, flags]}}: pwm.args.period = period, polarity per flags.
- [ ] AC-8: pwm_get / pwm_request_from_chip twice on same line: second returns -EBUSY (PWMF_REQUESTED).
- [ ] AC-9: pwm_apply_might_sleep with enabled=true ∧ period=0: -EINVAL (and pwm.state is unchanged unless current state also invalid).
- [ ] AC-10: pwm_apply_might_sleep with duty_cycle > period and current state valid: -EINVAL; with current state also invalid: accepted.
- [ ] AC-11: pwm_apply_might_sleep no-op when state already matches: returns 0 without invoking ops.apply / write_waveform.
- [ ] AC-12: pwm_apply_atomic on chip.atomic=false: WARN_ONCE.
- [ ] AC-13: pwm_round_waveform_might_sleep with period > S64_MAX: -EINVAL.
- [ ] AC-14: pwm_set_waveform_might_sleep(exact=true) when rounding occurs: -EDOM.
- [ ] AC-15: pwm_set_waveform_might_sleep(exact=false): rounding ack'd; returns 0.
- [ ] AC-16: pwm_get_waveform_might_sleep on driver without read_waveform: -EOPNOTSUPP.
- [ ] AC-17: pwm_capture without ops.capture: -ENOSYS.
- [ ] AC-18: pwm_put twice: second triggers pr_warn "PWM device already freed".
- [ ] AC-19: UAPI PWM_IOCTL_REQUEST + PWM_IOCTL_SETROUNDEDWF round-trip; close releases all requested lines.
- [ ] AC-20: sysfs export then unexport: PWMF_EXPORTED set/cleared; pwm device_register / device_unregister; suspend/resume snapshots and restores.

## Architecture

```
struct PwmChip {
  dev: Device,                       // class = pwm_class
  cdev: CharDev,                     // PWM_IOCTL_* UAPI
  ops: &'static PwmOps,
  owner: *Module,
  id: u32,                            // IDR-allocated
  npwm: u32,
  atomic: bool,
  uses_pwmchip_alloc: bool,
  operational: bool,                  // toggled under pwmchip-lock + pwm_lock
  atomic_lock: SpinLock,              // when atomic
  nonatomic_lock: Mutex,              // when !atomic
  of_xlate: Option<OfXlateFn>,
  gpio: Option<GpioChip>,             // CONFIG_PWM_PROVIDE_GPIO + write_waveform
  pwms: [PwmDevice; npwm],            // flex array
  // driver-private region trails pwms[]
}

struct PwmDevice {
  chip: NonNull<PwmChip>,
  hwpwm: u32,
  label: *const c_char,
  flags: AtomicBitmap,                // PWMF_REQUESTED, PWMF_EXPORTED
  args: PwmArgs { period, polarity },
  state: PwmState,
  last: PwmState,                     // PWM_DEBUG only
}

struct PwmState { period: u64, duty_cycle: u64, polarity: u8, enabled: bool, usage_power: bool }
struct PwmWaveform { period_length_ns: u64, duty_length_ns: u64, duty_offset_ns: u64 }
```

`Pwm::chip_add(chip, owner) -> Result<()>`:
1. Validate (chip, parent, ops, npwm, uses_pwmchip_alloc).
2. pwm_ops_check(chip).
3. chip.owner = owner; init atomic_lock or nonatomic_lock.
4. pwm_lock.lock():
   - chip.id = idr_alloc(&pwm_chips, chip).
   - dev_set_name("pwmchip{id}").
   - of_pwmchip_add (default of_xlate; of_node_get).
   - pwmchip_lock(chip): chip.operational = true.
   - If ops.write_waveform: chip.dev.devt = MKDEV(MAJOR(pwm_devt), id) (id < 256).
   - cdev_init(&chip.cdev, &pwm_cdev_fileops); cdev_device_add.
   - CONFIG_PWM_PROVIDE_GPIO ∧ ops.write_waveform: gpiochip_add_data(&chip.gpio, chip).

`Pwm::apply_might_sleep(pwm, state) -> Result<()>`:
1. might_sleep().
2. pwmchip-guard(chip).
3. Operational check.
4. If CONFIG_PWM_DEBUG ∧ chip.atomic: non_block_start/end around __pwm_apply.
5. else: __pwm_apply.

`Pwm::__pwm_apply(pwm, state)`:
1. pwm_state_valid(state): if invalid and current is also invalid: pwm.state = state; return Ok.
2. No-op fast path on equal-state.
3. If pwmchip_supports_waveform:
   - pwm_state2wf(state, &wf); round_waveform_tohw -> wfhw; (PWM_DEBUG: round_fromhw + check_rounding); write_waveform; pwm.state = state.
4. Else: ops.apply(chip, pwm, state); pwm.state = state; pwm_apply_debug (CONFIG_PWM_DEBUG).

`Pwm::round_waveform(pwm, wf) -> Result<i32>`:
1. BUG_ON(PWM_WFHWSIZE < ops.sizeof_wfhw).
2. pwmchip_supports_waveform check; pwm_wf_valid check.
3. pwmchip-guard; operational.
4. ret_tohw = ops.round_waveform_tohw(chip, pwm, wf, wfhw); ret_fromhw = ops.round_waveform_fromhw(chip, pwm, wfhw, wf).
5. Return ret_tohw (0 = exact; 1 = round-up).

`Pwm::set_waveform(pwm, wf, exact) -> Result<()>`:
1. might_sleep; pwmchip-guard; operational.
2. err = __pwm_set_waveform(pwm, wf, exact) (validate + round + write + state update + debug-readback).
3. err == -EDOM ⟹ -EINVAL; exact ∧ err == 1 ⟹ -EDOM; err == 1 ⟹ Ok; else err.

`Pwm::get(dev, con_id) -> Result<&PwmDevice>`:
1. fwnode = dev_fwnode(dev).
2. of_node: of_pwm_get(dev, to_of_node(fwnode), con_id).
3. acpi_node: acpi_pwm_get(fwnode); pass through ENOENT only.
4. else: scan pwm_lookup_list under pwm_lookup_lock; most-specific match (dev+con > dev > con); fallback request_module.
5. pwm_request_from_chip(chip, idx, label); pwm_device_link_add(dev, pwm).

`Pwm::request_from_chip(chip, index, label)` (pwm_lock held):
1. Validate index < chip.npwm.
2. test_bit(PWMF_REQUESTED): -EBUSY.
3. Operational check.
4. try_module_get(chip.owner); get_device(&chip.dev); ops.request (if any).
5. If ops.read_waveform ∨ ops.get_state: pwm_get_state_hw(pwm, &state); on Ok: pwm.state = state (PWM_DEBUG: pwm.last = state).
6. set_bit(PWMF_REQUESTED); pwm.label = label.

`Pwm::put(pwm)` (pwm_lock):
1. operational ∧ !test_and_clear_bit(PWMF_REQUESTED) ⟹ pr_warn double-free; return.
2. operational ∧ ops.free: ops.free(chip, pwm).
3. pwm.label = NULL; put_device(&chip.dev); module_put(chip.owner).

`Pwm::capture(pwm, result, timeout_ms)`:
1. ops.capture required else -ENOSYS.
2. mutex(pwm_lock); pwmchip-guard; operational.
3. ops.capture(chip, pwm, result, timeout_ms).

`Pwm::cdev_ioctl(file, cmd, arg)`:
- pwm_lock + operational check + dispatch on cmd; copy_from_user/copy_to_user for `struct pwmchip_waveform` with __pad sanity check.

`pwm_class_pm_ops`:
- suspend: for each exported pwm: snapshot to export.suspend; if enabled: disable; on error: roll back resume_npwm(i) for already-disabled.
- resume: restore export.suspend.enabled state.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `chip_alloc_size_overflow` | INVARIANT | per-pwmchip_alloc: alloc_size = ALIGN(struct_size(chip,pwms,npwm)) + sizeof_priv via size_add (overflow ⟹ -ENOMEM). |
| `chip_add_requires_uses_alloc` | INVARIANT | per-__pwmchip_add: !uses_pwmchip_alloc ⟹ -EINVAL. |
| `ops_check_waveform_or_apply` | INVARIANT | per-pwm_ops_check: exactly one of (write_waveform + round_to/from + size ok) or (apply present). |
| `wfhw_buffer_bounded` | INVARIANT | per-driver: sizeof_wfhw ≤ PWM_WFHWSIZE; BUG_ON triggered on overflow. |
| `pwm_state_valid_when_enabled` | INVARIANT | per-pwm_state_valid: enabled ⟹ period > 0 ∧ duty_cycle ≤ period. |
| `pwm_wf_valid_bounds` | INVARIANT | per-pwm_wf_valid: period ≤ S64_MAX ∧ duty ≤ period ∧ (offset == 0 ∨ offset < period). |
| `requested_flag_exclusive` | INVARIANT | per-pwm_device_request: PWMF_REQUESTED test+set under pwm_lock; double-request ⟹ -EBUSY. |
| `module_and_device_ref_balanced` | INVARIANT | per-request/put: try_module_get/get_device balanced by module_put/put_device. |
| `operational_gated_calls` | INVARIANT | per-ops invocation: chip.operational true at call time (pwmchip-locked). |
| `atomic_path_no_sleep` | INVARIANT | per-chip.atomic=true: pwm_apply_atomic accepted; pwm_apply_might_sleep under non_block_start/end in debug. |
| `set_waveform_exact_implies_no_round` | INVARIANT | per-pwm_set_waveform_might_sleep(exact=true): err == 1 ⟹ -EDOM. |
| `cdev_release_drains_pwms` | INVARIANT | per-pwm_cdev_release: every cdata.pwm[i] != NULL ⟹ pwm_put. |

### Layer 2: TLA+

`drivers/pwm/core.tla`:
- Per-chip lifecycle (alloc → add → operational → remove); per-pwm acquire/release; per-state-apply.
- Properties:
  - `safety_chip_not_used_after_remove` — per-pwmchip_remove: chip.operational == false ⟹ no further ops invoked.
  - `safety_pwm_single_consumer` — per-line: |{cl : PWMF_REQUESTED ∧ cl owns pwm}| ≤ 1.
  - `safety_no_op_on_unrequested` — per-pwm_apply / set_waveform / capture: PWMF_REQUESTED set.
  - `safety_state_matches_post_apply` — per-pwm_apply success: pwm.state == requested ∨ (invalid-to-invalid transition).
  - `safety_waveform_round_monotone` — per-round_waveform_tohw: period_length_ns / duty_length_ns / duty_offset_ns components weakly ≤ request when ret_tohw == 0 (pwm_check_rounding).
  - `liveness_get_eventually_resolves` — per-pwm_get: returns Ok ∨ -EPROBE_DEFER ∨ -ENODEV ∨ -ENOENT in bounded steps.
  - `liveness_pm_suspend_resume_idempotent` — per-pwm_class_pm: suspend(enabled)→resume restores enabled.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pwm::chip_alloc` post: chip.npwm == npwm ∧ chip.uses_pwmchip_alloc ∧ ¬chip.operational ∧ pwms[i].chip == chip ∧ pwms[i].hwpwm == i | `Pwm::chip_alloc` |
| `Pwm::chip_add` post: chip.operational ∧ chip.id ∈ pwm_chips ∧ cdev_device_add succeeded | `Pwm::chip_add` |
| `Pwm::__pwm_apply` post: pwm.state == *state (on Ok path) | `Pwm::__pwm_apply` |
| `Pwm::round_waveform` post: ret == 0 ⟹ pwm_check_rounding(wf_in, wf_out) | `Pwm::round_waveform` |
| `Pwm::set_waveform` post: exact ∧ Ok ⟹ pwmwfcmp(requested, applied) == 0 | `Pwm::set_waveform` |
| `Pwm::request_from_chip` post: Ok ⟹ PWMF_REQUESTED ∧ module_get succeeded ∧ get_device succeeded | `Pwm::request_from_chip` |
| `Pwm::put` post: ¬PWMF_REQUESTED ∧ module_put ∧ put_device | `Pwm::put` |
| `Pwm::cdev_ioctl` post: arg validated (__pad == 0) before pwm_lock release | `Pwm::cdev_ioctl` |

### Layer 4: Verus/Creusot functional

`Per-chip_alloc → __pwmchip_add → idr_alloc → cdev_device_add → operational; per-consumer pwm_get (DT/ACPI/lookup) → pwm_request_from_chip (PWMF_REQUESTED, module+device ref); per-pwm_apply_might_sleep → pwm_state_valid → state2wf+round_to+write_waveform XOR ops.apply → pwm.state mutation; per-pwm_set_waveform(exact) round-trip pwmwfcmp invariance; per-pwm_class suspend/resume idempotence` semantic equivalence: per-Documentation/driver-api/pwm.rst + per-pwm consumer/provider tests in tools/testing/selftests/pwm/.

## Hardening

(Inherits row-1 features from `drivers/pwm/00-overview.md` § Hardening.)

PWM-core reinforcement:

- **Per-pwm_lock + per-chip pwmchip_lock(mutex|spin) discipline** — defense against per-races between consumer apply, sysfs writers, UAPI ioctl, and pwmchip_remove.
- **Per-chip.atomic chooses spin-lock vs mutex** — defense against per-sleep-in-atomic deadlock for fast PWM apply (LED dimming under IRQ).
- **Per-uses_pwmchip_alloc required at register-time** — defense against per-stack-allocated chip (embedded device disappears on driver detach, dangling cdev).
- **Per-operational gate on every ops dispatch** — defense against per-use-after-pwmchip_remove.
- **Per-PWM_WFHWSIZE BUG_ON when driver lies about sizeof_wfhw** — defense against per-stack-overrun in opaque wfhw buffer.
- **Per-pwm_wf_valid (period ≤ S64_MAX)** — defense against per-signed-overflow downstream and reserves negative encoding for future inverted polarity.
- **Per-pwm_state_valid (enabled ⟹ period > 0 ∧ duty ≤ period)** — defense against per-driver-divide-by-zero or duty > 100% transients.
- **Per-PWMF_REQUESTED test_and_set under pwm_lock** — defense against per-double-request and per-double-free (pwm_put double-call warns).
- **Per-module_get + device_get balanced get/put** — defense against per-driver-unload while consumer alive and per-released device descriptor UAF.
- **Per-pwm_apply no-op fast path** — defense against per-bus-thrash from idempotent re-apply (e.g. backlight curve animations).
- **Per-pwm_apply_debug readback + reapply idempotency** — defense against per-driver `.apply` non-idempotency / silent rounding bugs (CONFIG_PWM_DEBUG).
- **Per-non_block_start/non_block_end around atomic-chip apply** — defense against per-driver hidden sleep when chip.atomic asserted.
- **Per-pwm_class_pm rollback on suspend failure** — defense against per-partial-suspend leaving some lines stuck enabled.
- **Per-UAPI __pad sanity check (must be 0)** — defense against per-future-flag-leak via uninitialized userspace padding.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Individual PWM controller drivers (pwm-imx, pwm-rockchip, pwm-stm32, pwm-bcm2835, pwm-cros-ec, etc.) — each gets its own Tier-3 if/when expanded.
- PWM consumer drivers (drivers/video/backlight/pwm_bl.c, drivers/hwmon/pwm-fan.c, drivers/leds/leds-pwm.c) — separate Tier-3 if expanded.
- DT bindings format (#pwm-cells, pwms / pwm-names) — Documentation/devicetree/bindings/pwm/.
- ACPI PWM bindings — covered by ACPI subsystem docs.
- sysfs-class semantics minutiae — Documentation/ABI/testing/sysfs-class-pwm.
- tools/testing/selftests/pwm/ harness.
- Implementation code.
