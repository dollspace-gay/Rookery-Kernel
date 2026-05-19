# Tier-3: drivers/regulator/fixed.c — Fixed-voltage regulator driver

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/regulator/00-overview.md
upstream-paths:
  - drivers/regulator/fixed.c (~402 lines)
  - include/linux/regulator/fixed.h
  - include/linux/regulator/driver.h
  - Documentation/devicetree/bindings/regulator/fixed-regulator.yaml
-->

## Summary

The **fixed-voltage regulator** driver implements the simplest possible regulator: a power rail providing a single, constant output voltage that may be gated on/off through an optional GPIO enable line, gated clock, or generic-PM-domain performance state. The driver instantiates one `struct regulator_dev` per `regulator-fixed` (or `regulator-fixed-clock` / `regulator-fixed-domain`) devicetree node and registers it with the regulator core via `devm_regulator_register`. Per-`struct fixed_voltage_data`: private per-instance state holding the `regulator_desc`, the back-reference `regulator_dev`, an optional `enable_clock`, an `enable_counter`, and (for domain mode) the `performance_state`. Per-`struct fixed_voltage_config`: configuration parsed from devicetree (`supply_name`, `microvolts`, `startup_delay`, `off_on_delay`, `enabled_at_boot`, `input_supply`, `init_data`). Per-`reg_fixed_voltage_probe`: validates min_uV == max_uV (otherwise EINVAL), populates ops based on which `fixed_dev_type` matched (no-op / clock-gated / domain-perf), pulls the optional GPIO via `gpiod_get_optional` with `GPIOD_FLAGS_BIT_NONEXCLUSIVE` (so multiple regulators can share an enable line), and registers the regulator. Per-optional under-voltage IRQ: when firmware provides an interrupt the driver wires a threaded handler that calls `regulator_notifier_call_chain(REGULATOR_EVENT_UNDER_VOLTAGE)`. Critical for: board power-on, supplying always-on rails that nevertheless need a consumer-visible regulator object, mixed-controllable test rigs, simple GPIO-gated LDOs.

This Tier-3 covers `drivers/regulator/fixed.c` (~402 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct fixed_voltage_data` | per-instance private state | `FixedVoltageData` |
| `struct fixed_voltage_config` | per-platform/of config | `FixedVoltageConfig` |
| `struct fixed_dev_type` | per-compatible match-data | `FixedDevType` |
| `reg_fixed_voltage_probe()` | per-platform-device probe | `FixedRegulator::probe` |
| `of_get_fixed_voltage_config()` | per-of parse | `FixedRegulator::parse_of` |
| `reg_fixed_get_irqs()` | per-optional UV IRQ wiring | `FixedRegulator::get_irqs` |
| `reg_fixed_under_voltage_irq_handler()` | per-UV-IRQ handler | `FixedRegulator::uv_irq_handler` |
| `reg_clock_enable()` / `reg_clock_disable()` | per-clock-gated enable/disable | `FixedRegulator::clock_enable` / `clock_disable` |
| `reg_domain_enable()` / `reg_domain_disable()` | per-genpd-perf enable/disable | `FixedRegulator::domain_enable` / `domain_disable` |
| `reg_is_enabled()` | per-state query | `FixedRegulator::is_enabled` |
| `fixed_voltage_ops` | no-op ops (always-on, GPIO-only) | `FIXED_VOLTAGE_OPS` |
| `fixed_voltage_clkenabled_ops` | clock-gated ops | `FIXED_VOLTAGE_CLKENABLED_OPS` |
| `fixed_voltage_domain_ops` | genpd-domain ops | `FIXED_VOLTAGE_DOMAIN_OPS` |
| `fixed_of_match[]` | per-compatible table | `FIXED_OF_MATCH` |
| `regulator_fixed_voltage_driver` | per-platform_driver | `REGULATOR_FIXED_VOLTAGE_DRIVER` |
| `regulator-fixed` / `-clock` / `-domain` | per-DT compatible strings | shared |

## Compatibility contract

REQ-1: struct fixed_voltage_data (per-instance):
- desc: per-`regulator_desc` (name, type=REGULATOR_VOLTAGE, owner, ops, n_voltages, fixed_uV, enable_time, off_on_delay, supply_name).
- dev: per-back-pointer to registered `regulator_dev`.
- enable_clock: per-optional `struct clk *` for `regulator-fixed-clock`.
- enable_counter: per-software refcount (incremented on enable, decremented on disable; backs `is_enabled`).
- performance_state: per-domain-mode required OPP performance state (>= 0).

REQ-2: struct fixed_voltage_config (per-OF / per-platform-data):
- supply_name: per-rail name (mirrors `init_data->constraints.name`).
- input_supply: per-upstream supply name (`"vin"` if `vin-supply` is present in DT).
- microvolts: per-output voltage; must equal `min_uV == max_uV` from constraints.
- startup_delay: per-`startup-delay-us` (DT) — fed to `desc.enable_time`.
- off_on_delay: per-`off-on-delay-us` (DT) — fed to `desc.off_on_delay`.
- enabled_at_boot: per-`init_data->constraints.boot_on`. Drives initial GPIO state (HIGH if true).
- gpio_is_open_drain: (legacy field from header).
- init_data: per-`regulator_init_data` parsed via `of_get_regulator_init_data`.

REQ-3: struct fixed_dev_type (per-compatible match-data):
- has_enable_clock: per-`regulator-fixed-clock` flag.
- has_performance_state: per-`regulator-fixed-domain` flag.

REQ-4: of_get_fixed_voltage_config(dev, desc):
- /* devm-allocate config */
- config = devm_kzalloc(dev, sizeof(*config), GFP_KERNEL); if !config: return ERR_PTR(-ENOMEM).
- /* Parse standard regulator init_data */
- config.init_data = of_get_regulator_init_data(dev, np, desc); if !init_data: return ERR_PTR(-EINVAL).
- init_data.constraints.apply_uV = 0.
- config.supply_name = init_data.constraints.name.
- /* Enforce fixed voltage */
- if init_data.constraints.min_uV == init_data.constraints.max_uV:
  - config.microvolts = init_data.constraints.min_uV.
- else: dev_err("Fixed regulator specified with variable voltages"); return ERR_PTR(-EINVAL).
- if init_data.constraints.boot_on: config.enabled_at_boot = true.
- of_property_read_u32(np, "startup-delay-us", &config.startup_delay).
- of_property_read_u32(np, "off-on-delay-us", &config.off_on_delay).
- if of_property_present(np, "vin-supply"): config.input_supply = "vin".
- return config.

REQ-5: reg_clock_enable / reg_clock_disable (clock-gated mode):
- enable: clk_prepare_enable(priv.enable_clock); if !ret: priv.enable_counter++.
- disable: clk_disable_unprepare(priv.enable_clock); priv.enable_counter--.
- Errors from clk_prepare_enable propagate up.

REQ-6: reg_domain_enable / reg_domain_disable (genpd-perf mode):
- enable: dev_pm_genpd_set_performance_state(parent, priv.performance_state); if !ret: priv.enable_counter++.
- disable: dev_pm_genpd_set_performance_state(parent, 0); priv.enable_counter--.

REQ-7: reg_is_enabled:
- Returns priv.enable_counter > 0 (software-tracked state; the GPIO-driven default path uses the regulator core's GPIO-enable bookkeeping instead).

REQ-8: reg_fixed_under_voltage_irq_handler (threaded IRQ):
- regulator_notifier_call_chain(priv.dev, REGULATOR_EVENT_UNDER_VOLTAGE, NULL).
- return IRQ_HANDLED.

REQ-9: reg_fixed_get_irqs:
- ret = fwnode_irq_get(dev_fwnode(dev), 0).
- if ret == -EINVAL: return 0 (IRQ is optional).
- if ret < 0: return dev_err_probe(dev, ret, "Failed to get IRQ").
- devm_request_threaded_irq(dev, ret, NULL, reg_fixed_under_voltage_irq_handler, IRQF_ONESHOT, "under-voltage", priv).
- if err: return dev_err_probe(dev, ret, "Failed to request IRQ").
- return 0.

REQ-10: reg_fixed_voltage_probe high-level flow:
- /* Allocate per-instance state */
- drvdata = devm_kzalloc(&pdev->dev, sizeof(*drvdata), GFP_KERNEL); if !drvdata: -ENOMEM.
- /* Parse config: OF vs platform_data */
- if pdev.dev.of_node: config = of_get_fixed_voltage_config(&pdev->dev, &drvdata->desc); if IS_ERR(config): return PTR_ERR(config).
- else: config = dev_get_platdata(&pdev->dev).
- if !config: return -ENOMEM.
- /* Populate desc */
- drvdata.desc.name = devm_kstrdup(supply_name); if !name: return -ENOMEM.
- drvdata.desc.type = REGULATOR_VOLTAGE.
- drvdata.desc.owner = THIS_MODULE.
- /* Select ops based on compatible */
- if drvtype && has_enable_clock: ops = &fixed_voltage_clkenabled_ops; drvdata.enable_clock = devm_clk_get(dev, NULL); IS_ERR: return PTR_ERR.
- else if drvtype && has_performance_state: ops = &fixed_voltage_domain_ops; drvdata.performance_state = of_get_required_opp_performance_state(np, 0); <0: return err.
- else: ops = &fixed_voltage_ops (no-op).
- desc.enable_time = config.startup_delay; desc.off_on_delay = config.off_on_delay.
- if config.input_supply: desc.supply_name = devm_kstrdup(input_supply); if !: -ENOMEM.
- if config.microvolts: desc.n_voltages = 1.
- desc.fixed_uV = config.microvolts.

REQ-11: Probe GPIO acquisition:
- gflags = config.enabled_at_boot ? GPIOD_OUT_HIGH : GPIOD_OUT_LOW.
- gflags |= GPIOD_FLAGS_BIT_NONEXCLUSIVE (allows shared enable line between two regulators; only first call applies inversion/open-drain attributes).
- cfg.ena_gpiod = gpiod_get_optional(&pdev->dev, NULL, gflags) (NOT devm_; regulator core takes ownership of lifetime).
- if IS_ERR: return dev_err_probe.

REQ-12: Probe registration:
- cfg.dev = &pdev->dev.
- cfg.init_data = config.init_data.
- cfg.driver_data = drvdata.
- cfg.of_node = pdev.dev.of_node.
- drvdata.dev = devm_regulator_register(&pdev->dev, &drvdata->desc, &cfg).
- if IS_ERR: return dev_err_probe("Failed to register regulator: %ld").
- platform_set_drvdata(pdev, drvdata).
- dev_dbg("%s supplying %duV").
- ret = reg_fixed_get_irqs(dev, drvdata); if ret: return ret.
- return 0.

REQ-13: regulator_ops variants:
- fixed_voltage_ops: empty (all-default; GPIO-driven enable/disable lives in regulator core; pure-software always-on if no GPIO).
- fixed_voltage_clkenabled_ops: { enable=reg_clock_enable, disable=reg_clock_disable, is_enabled=reg_is_enabled }.
- fixed_voltage_domain_ops: { enable=reg_domain_enable, disable=reg_domain_disable, is_enabled=reg_is_enabled }.

REQ-14: Devicetree match table fixed_of_match[]:
- "regulator-fixed" → &fixed_voltage_data (has_enable_clock=false, has_performance_state=false).
- "regulator-fixed-clock" → &fixed_clkenable_data (has_enable_clock=true).
- "regulator-fixed-domain" → &fixed_domain_data (has_performance_state=true).
- Sentinel { } terminator.
- MODULE_DEVICE_TABLE(of, fixed_of_match).

REQ-15: platform_driver regulator_fixed_voltage_driver:
- .probe = reg_fixed_voltage_probe.
- .driver.name = "reg-fixed-voltage".
- .driver.probe_type = PROBE_PREFER_ASYNCHRONOUS.
- .driver.of_match_table = of_match_ptr(fixed_of_match).
- Registered via subsys_initcall (so regulators are available for arbitrary-subsystem clients during normal device-probe).

REQ-16: GPIO-shared-enable semantics:
- GPIOD_FLAGS_BIT_NONEXCLUSIVE permits two `regulator-fixed` nodes to reference the same GPIO line.
- Only the first probe call initializes flags (inversion / open-drain); subsequent callers must assume identical attributes.
- Marked FIXME in upstream — a known limitation, not a bug.

REQ-17: variable-voltage rejection:
- If init_data.constraints.min_uV != max_uV the driver refuses to bind (EINVAL).
- Guarantees that a registered fixed regulator never reports more than one voltage value.

## Acceptance Criteria

- [ ] AC-1: Probe with `compatible = "regulator-fixed"`, min_uV=3300000, max_uV=3300000: drvdata.desc.fixed_uV == 3300000, n_voltages == 1, ops == fixed_voltage_ops.
- [ ] AC-2: Probe with min_uV != max_uV: returns -EINVAL with `dev_err("Fixed regulator specified with variable voltages")`.
- [ ] AC-3: Probe with `compatible = "regulator-fixed-clock"`: devm_clk_get acquires clock; ops == fixed_voltage_clkenabled_ops; enable raises clk_prepare_enable + enable_counter.
- [ ] AC-4: Probe with `compatible = "regulator-fixed-domain"`: of_get_required_opp_performance_state succeeds; ops == fixed_voltage_domain_ops.
- [ ] AC-5: `regulator-boot-on` in DT: enabled_at_boot=true → gflags = GPIOD_OUT_HIGH at GPIO acquisition.
- [ ] AC-6: `gpios = <&gpio …>` present: cfg.ena_gpiod populated via gpiod_get_optional with GPIOD_FLAGS_BIT_NONEXCLUSIVE; regulator core owns lifetime (not devm).
- [ ] AC-7: Two `regulator-fixed` nodes sharing a GPIO: both probe successfully (NONEXCLUSIVE).
- [ ] AC-8: `startup-delay-us = <500>` in DT: desc.enable_time == 500.
- [ ] AC-9: `off-on-delay-us = <1000>` in DT: desc.off_on_delay == 1000.
- [ ] AC-10: `vin-supply = <&parent>`: desc.supply_name == "vin".
- [ ] AC-11: Optional interrupt absent (fwnode_irq_get == -EINVAL): probe still succeeds; no IRQ wired.
- [ ] AC-12: Optional interrupt present: devm_request_threaded_irq succeeds; firing the IRQ invokes regulator_notifier_call_chain(REGULATOR_EVENT_UNDER_VOLTAGE).
- [ ] AC-13: reg_is_enabled returns enable_counter > 0 after balanced enable/disable.
- [ ] AC-14: of_get_required_opp_performance_state failure in domain mode: probe returns negative error.
- [ ] AC-15: dev_get_platdata == NULL with no of_node: probe returns -ENOMEM.

## Architecture

```
struct FixedVoltageData {
  desc: RegulatorDesc,
  dev: *RegulatorDev,                  // back-pointer
  enable_clock: Option<*Clk>,
  enable_counter: u32,
  performance_state: i32,
}

struct FixedVoltageConfig {
  supply_name: &'static str,
  input_supply: Option<&'static str>,
  microvolts: i32,
  startup_delay: u32,
  off_on_delay: u32,
  enabled_at_boot: bool,
  init_data: *RegulatorInitData,
}

struct FixedDevType {
  has_enable_clock: bool,
  has_performance_state: bool,
}
```

`FixedRegulator::probe(pdev) -> Result<(), Errno>`:
1. drvdata = devm_kzalloc(&pdev.dev, FixedVoltageData).
2. /* Parse config */
3. config = if pdev.dev.of_node.is_some():
   - parse_of(&pdev.dev, &drvdata.desc)?
   else:
   - dev_get_platdata(&pdev.dev).ok_or(-ENOMEM)?
4. /* Build desc */
5. drvdata.desc.name = devm_kstrdup(config.supply_name)?.
6. drvdata.desc.type = REGULATOR_VOLTAGE.
7. drvdata.desc.owner = THIS_MODULE.
8. drvtype = of_device_get_match_data(&pdev.dev) as &FixedDevType.
9. /* Select ops */
10. if drvtype.has_enable_clock:
    - drvdata.desc.ops = &FIXED_VOLTAGE_CLKENABLED_OPS.
    - drvdata.enable_clock = devm_clk_get(dev, None)?.
11. else if drvtype.has_performance_state:
    - drvdata.desc.ops = &FIXED_VOLTAGE_DOMAIN_OPS.
    - drvdata.performance_state = of_get_required_opp_performance_state(np, 0)?.
12. else:
    - drvdata.desc.ops = &FIXED_VOLTAGE_OPS.
13. drvdata.desc.enable_time = config.startup_delay.
14. drvdata.desc.off_on_delay = config.off_on_delay.
15. if let Some(vin) = config.input_supply:
    - drvdata.desc.supply_name = devm_kstrdup(vin)?.
16. if config.microvolts != 0: drvdata.desc.n_voltages = 1.
17. drvdata.desc.fixed_uV = config.microvolts.
18. /* GPIO */
19. let gflags = if config.enabled_at_boot { GPIOD_OUT_HIGH } else { GPIOD_OUT_LOW } | GPIOD_FLAGS_BIT_NONEXCLUSIVE.
20. cfg.ena_gpiod = gpiod_get_optional(&pdev.dev, None, gflags)?.
21. cfg.dev = &pdev.dev; cfg.init_data = config.init_data; cfg.driver_data = drvdata; cfg.of_node = pdev.dev.of_node.
22. /* Register */
23. drvdata.dev = devm_regulator_register(&pdev.dev, &drvdata.desc, &cfg)?.
24. platform_set_drvdata(pdev, drvdata).
25. /* Optional UV IRQ */
26. FixedRegulator::get_irqs(&pdev.dev, drvdata)?.
27. return Ok(()).

`FixedRegulator::parse_of(dev, desc) -> Result<*FixedVoltageConfig, Errno>`:
1. config = devm_kzalloc(dev, FixedVoltageConfig)?.
2. config.init_data = of_get_regulator_init_data(dev, dev.of_node, desc).ok_or(-EINVAL)?.
3. config.init_data.constraints.apply_uV = 0.
4. config.supply_name = config.init_data.constraints.name.
5. if config.init_data.constraints.min_uV == config.init_data.constraints.max_uV:
   - config.microvolts = config.init_data.constraints.min_uV.
6. else:
   - dev_err("Fixed regulator specified with variable voltages").
   - return Err(-EINVAL).
7. if config.init_data.constraints.boot_on: config.enabled_at_boot = true.
8. of_property_read_u32(np, "startup-delay-us", &config.startup_delay).
9. of_property_read_u32(np, "off-on-delay-us", &config.off_on_delay).
10. if of_property_present(np, "vin-supply"): config.input_supply = Some("vin").
11. return Ok(config).

`FixedRegulator::clock_enable(rdev) -> Result<(), Errno>`:
1. priv = rdev_get_drvdata(rdev) as &mut FixedVoltageData.
2. clk_prepare_enable(priv.enable_clock)?.
3. priv.enable_counter += 1.
4. return Ok(()).

`FixedRegulator::clock_disable(rdev) -> Result<(), Errno>`:
1. priv = rdev_get_drvdata(rdev).
2. clk_disable_unprepare(priv.enable_clock).
3. priv.enable_counter -= 1.
4. return Ok(()).

`FixedRegulator::domain_enable(rdev) -> Result<(), Errno>`:
1. priv = rdev_get_drvdata(rdev).
2. dev_pm_genpd_set_performance_state(rdev.dev.parent, priv.performance_state)?.
3. priv.enable_counter += 1.
4. return Ok(()).

`FixedRegulator::domain_disable(rdev) -> Result<(), Errno>`:
1. priv = rdev_get_drvdata(rdev).
2. dev_pm_genpd_set_performance_state(rdev.dev.parent, 0)?.
3. priv.enable_counter -= 1.
4. return Ok(()).

`FixedRegulator::is_enabled(rdev) -> i32`:
1. priv = rdev_get_drvdata(rdev).
2. return (priv.enable_counter > 0) as i32.

`FixedRegulator::uv_irq_handler(irq, data) -> IrqReturn`:
1. priv = data as &FixedVoltageData.
2. regulator_notifier_call_chain(priv.dev, REGULATOR_EVENT_UNDER_VOLTAGE, NULL).
3. return IRQ_HANDLED.

`FixedRegulator::get_irqs(dev, priv) -> Result<(), Errno>`:
1. ret = fwnode_irq_get(dev_fwnode(dev), 0).
2. if ret == -EINVAL: return Ok(()).        // optional IRQ
3. if ret < 0: return dev_err_probe(dev, ret, "Failed to get IRQ").
4. devm_request_threaded_irq(dev, ret, NULL, uv_irq_handler, IRQF_ONESHOT, "under-voltage", priv)?.
5. return Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `probe_rejects_variable_voltage` | INVARIANT | per-parse_of: min_uV != max_uV ⟹ -EINVAL. |
| `desc_n_voltages_one_when_microvolts_set` | INVARIANT | per-probe: config.microvolts != 0 ⟹ desc.n_voltages == 1. |
| `ena_gpiod_not_devm_managed` | INVARIANT | per-probe: gpiod_get_optional ≠ devm_*; regulator core releases. |
| `clk_enable_path_increments_counter` | INVARIANT | per-clock_enable: success ⟹ enable_counter += 1. |
| `clk_disable_path_decrements_counter` | INVARIANT | per-clock_disable: enable_counter -= 1 unconditionally. |
| `is_enabled_reflects_counter` | INVARIANT | per-is_enabled: returns enable_counter > 0. |
| `optional_irq_absent_is_success` | INVARIANT | per-get_irqs: fwnode_irq_get == -EINVAL ⟹ Ok(()). |
| `nonexclusive_flag_always_set_for_gpio` | INVARIANT | per-probe: gflags & GPIOD_FLAGS_BIT_NONEXCLUSIVE. |

### Layer 2: TLA+

`drivers/regulator/fixed.tla`:
- Per-probe + per-config-parse + per-ops-selection + per-enable/disable + per-UV-IRQ.
- Properties:
  - `safety_no_variable_voltage_registers` — per-probe: registered regulator has exactly one voltage.
  - `safety_compatible_selects_correct_ops` — per-compatible string ⟹ correct ops vtable.
  - `safety_enable_counter_balanced` — per-enable/disable: balanced sequences ⟹ counter returns to 0.
  - `safety_irq_optional` — per-fwnode_irq_get == -EINVAL: probe still succeeds.
  - `liveness_probe_eventually_registers_or_errors` — per-probe: terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `FixedRegulator::probe` post: ret==0 ⟹ drvdata.dev registered ∧ desc.fixed_uV == config.microvolts | `FixedRegulator::probe` |
| `FixedRegulator::parse_of` post: returned config has min_uV == max_uV | `FixedRegulator::parse_of` |
| `FixedRegulator::clock_enable` post: prepare-enable succeeded ⟹ counter+=1 | `FixedRegulator::clock_enable` |
| `FixedRegulator::domain_enable` post: perf-state set ⟹ counter+=1 | `FixedRegulator::domain_enable` |
| `FixedRegulator::is_enabled` post: returns (counter>0) as i32 | `FixedRegulator::is_enabled` |
| `FixedRegulator::get_irqs` post: -EINVAL coerced to Ok | `FixedRegulator::get_irqs` |

### Layer 4: Verus/Creusot functional

`Per-DT node → parse_of → ops-select(compat) → gpiod_get_optional(NONEXCLUSIVE) → devm_regulator_register → optional fwnode_irq_get + devm_request_threaded_irq` semantic equivalence: per-Documentation/devicetree/bindings/regulator/fixed-regulator.yaml + per-include/linux/regulator/fixed.h.

## Hardening

(Inherits row-1 features from `drivers/regulator/00-overview.md` § Hardening.)

Fixed-voltage reinforcement:

- **Per-min_uV == max_uV enforced** — defense against per-mis-binding a tunable regulator as fixed.
- **Per-GPIOD_FLAGS_BIT_NONEXCLUSIVE explicit** — defense against per-shared-line probe rejection.
- **Per-ena_gpiod NOT devm_ owned** — defense against per-double-free when regulator core releases.
- **Per-startup-delay honored by core** — defense against per-consumer-reads-before-rail-settles.
- **Per-off-on-delay honored** — defense against per-rapid-cycle out-of-spec stress.
- **Per-fwnode_irq_get -EINVAL = absent (not error)** — defense against per-optional-IRQ probe failure.
- **Per-IRQF_ONESHOT for threaded UV handler** — defense against per-re-entrant notifier walk.
- **Per-devm_kzalloc for drvdata** — defense against per-leak-on-probe-error.
- **Per-devm_kstrdup for supply names** — defense against per-stale-DT-pointer reads.
- **Per-of_get_required_opp_performance_state failure aborts probe** — defense against per-uninitialized perf-state.
- **Per-PROBE_PREFER_ASYNCHRONOUS** — defense against per-deferred-probe deadlock with consumer rails.
- **Per-dev_err_probe (not dev_err)** — defense against per-deferred-probe log flood.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — /sys/class/regulator/regulator.N/* attrs reflected for a fixed-regulator instance whitelisted; defense against per-oversized-sysfs-read.
- **PAX_KERNEXEC** — reg_fixed_voltage_probe, fixed_voltage_enable / fixed_voltage_disable, fixed_voltage_is_enabled run W^X.
- **PAX_RANDKSTACK** — per-/sys/class/regulator-write and per-GPIO-IRQ (status / enable-active-low edge) entry randomize kernel-stack offset.
- **PAX_REFCOUNT** — struct regulator_dev, struct fixed_voltage_data refs saturating refcount_t; defense against per-deferred-probe loop refcount overflow.
- **PAX_MEMORY_SANITIZE** — fixed_voltage_data, fixed_voltage_config, regulator_init_data slabs poison-on-free; defense against per-prior-DT-config leak across re-bind.
- **PAX_UDEREF** — fixed regulator exposes no direct UAPI beyond the regulator-core sysfs; copy_*_user paths enforce split address spaces at the parent layer.
- **PAX_RAP/kCFI** — fixed_voltage_ops vtable (enable, disable, is_enabled, get_voltage) CFI-protected via PAX_RAP / kCFI.
- **GRKERNSEC_HIDESYM** — fixed_voltage_ops, reg_fixed_voltage_driver addresses hidden from /proc/kallsyms.
- **GRKERNSEC_DMESG** — "fixed-regulator: GPIO request failed", "fixed-regulator: failed to register" prints restricted.
- **GPIO-enable CAP_SYS_RAWIO** — the underlying enable-GPIO line (gpiod_set_value_cansleep inside fixed_voltage_enable) is reached only via the regulator-core consumer API, which itself sits behind /sys/class/regulator (CAP_SYS_ADMIN); direct GPIO sysfs access to the same line (/sys/class/gpio or /dev/gpiochipN) gated on CAP_SYS_RAWIO; defense against per-unprivileged-GPIO-write bypassing the regulator-core consumer count and toggling a downstream rail.
- **Per-PROBE_PREFER_ASYNCHRONOUS** — defense against per-deferred-probe deadlock with consumer rails.
- **Per-dev_err_probe (not dev_err)** — defense against per-deferred-probe log flood.
- **Per-fixed-voltage no-set_voltage** — fixed_voltage_ops deliberately omits ->set_voltage; defense against per-mis-binding a fixed rail and trying to drive it to an unsupported value.
- **Per-startup-delay enforced** — fixed_voltage_config->startup_delay observed after enable; defense against per-consumer-using-rail-before-stable causing brownout fault on a downstream device.
- **Per-enable-active-low polarity respected** — GPIO logical level set per fixed_voltage_config->enable_active_high; defense against per-DT-typo enabling rail by default.
- Rationale: fixed regulator is a thin GPIO-or-always-on wrapper around the regulator core; grsec posture combines CAP_SYS_ADMIN at the regulator-core sysfs, CAP_SYS_RAWIO at any sibling /sys/class/gpio path that could bypass it, CFI on fixed_voltage_ops, startup-delay and polarity discipline, and sanitize-on-free of the fixed_voltage_data slab.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Regulator core registration/lifecycle (covered in `drivers/regulator/core.md` Tier-3)
- Regmap helper ops (covered in `drivers/regulator/helpers.md` Tier-3)
- GPIO descriptor lookup (covered in `drivers/gpio/gpiolib.md` Tier-3)
- Generic PM domain performance state plumbing (covered in `drivers/base/power/domain.md` Tier-3 if expanded)
- clk subsystem prepare/enable (covered in `drivers/clk/clk.md` Tier-3 if expanded)
- Devicetree property parsing core (covered in `drivers/of/base.md` Tier-3 if expanded)
- Implementation code
