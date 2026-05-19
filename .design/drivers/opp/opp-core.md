# Tier-3: drivers/opp/{core,of,cpu}.c — Operating Performance Points (OPP) tables, freq/voltage scaling, devicetree binding, per-CPU cpufreq glue

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/opp/core.c
  - drivers/opp/of.c
  - drivers/opp/cpu.c
  - drivers/opp/opp.h
  - drivers/opp/debugfs.c
  - include/linux/pm_opp.h
-->

## Summary

The OPP (Operating Performance Points) framework is the kernel's canonical model for "tuple of {frequency, voltage(s), bandwidth, performance state}" for a clocked device. A `struct opp_table` is attached to a `struct device` (CPU, GPU, NPU, devfreq client) and contains an ordered list of `struct dev_pm_opp`. Drivers — cpufreq governors, devfreq, and SoC clock controllers — call `dev_pm_opp_set_rate()` which (a) finds the matching OPP, (b) ramps regulators on the supplies, (c) calls the `config_clks` callback to actually program the clock, (d) propagates required-OPPs to genpd performance states, and (e) updates `interconnect` bandwidth votes. `core.c` is the device-API + lookup engine; `of.c` parses the devicetree `operating-points-v2` binding (per-OPP `opp-hz`, `opp-microvolt`, `opp-microamp`, `opp-level`, `opp-peak-kBps`, `required-opps`, supported-hw mask); `cpu.c` glues OPP tables into the cpufreq `freq_table` and per-policy cpumask sharing. The framework also publishes Energy Model entries (`dev_pm_opp_of_register_em`) which the scheduler EAS uses for power-aware task placement.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct opp_table` | per-device OPP container, refcounted, registered against `dev` | `drivers::opp::OppTable` |
| `struct dev_pm_opp` | per-OPP record (key, supplies[], bandwidth[], required_opps[]) | `drivers::opp::Opp` |
| `dev_pm_opp_get_opp_table(dev)` / `_put_opp_table(table)` | acquire/release per-device table | `OppTable::get` / `_put` |
| `dev_pm_opp_add_dynamic(dev, data)` / `_remove(dev, freq)` | programmatic OPP add/remove | `OppTable::add_dynamic` / `_remove` |
| `dev_pm_opp_find_freq_exact(dev, freq, avail)` / `_ceil` / `_floor` | lookup helpers | `OppTable::find_freq_*` |
| `dev_pm_opp_find_level_*` / `_find_bw_*` | level- and bandwidth-keyed lookups | `OppTable::find_level_*` |
| `dev_pm_opp_set_rate(dev, target_freq)` | full apply pipeline (regulator, clk, genpd, icc) | `OppTable::set_rate` |
| `dev_pm_opp_set_opp(dev, opp)` | apply a specific OPP object directly | `OppTable::set_opp` |
| `dev_pm_opp_set_config(dev, config)` / `dev_pm_opp_clear_config(token)` | per-device callback hooks (config_clks, config_regulators, supported_hw, etc.) | `OppTable::set_config` |
| `dev_pm_opp_adjust_voltage(dev, freq, u_volt, u_volt_min, u_volt_max)` | retune voltage for an existing OPP | `OppTable::adjust_voltage` |
| `dev_pm_opp_enable(dev, freq)` / `_disable(...)` | per-OPP enable bit | `Opp::enable` / `_disable` |
| `dev_pm_opp_of_add_table(dev)` / `_of_remove_table(dev)` | parse devicetree `operating-points-v2` for `dev` | `OfBinding::add_table` / `_remove_table` |
| `_of_init_opp_table(table, dev, opp_np)` | OPP-v2 binding parser entry | `OfBinding::init_opp_table` |
| `opp_parse_supplies(opp, dev, opp_table)` | `opp-microvolt[-<name>]` + `opp-microamp[-<name>]` parsing | `OfBinding::parse_supplies` |
| `dev_pm_opp_of_register_em(dev, cpus)` | register Energy Model for EAS | `OfBinding::register_em` |
| `dev_pm_opp_of_cpumask_add_table(cpumask)` / `_get_sharing_cpus(...)` | per-CPU OPP table install + sharing-mask query | `OfBinding::cpumask_add_table` / `_get_sharing_cpus` |
| `dev_pm_opp_init_cpufreq_table(dev, &table)` / `_free_cpufreq_table(...)` | export OPP table into cpufreq `freq_table` form | `CpuBinding::init_cpufreq_table` / `_free_cpufreq_table` |
| `_set_required_opps(dev, table, opp, scaling_up)` | walk `required-opps` chain, propagate to genpd | `OppTable::set_required_opps` |
| `_set_opp_bw(table, opp, dev)` | interconnect-path bandwidth vote per OPP | `OppTable::set_bw` |
| `_set_opp_voltage(dev, reg, supply)` | regulator setting with min/max bound | `OppTable::set_voltage` |

## Compatibility contract

REQ-1: Per-device `opp_table` is allocated lazily on first `dev_pm_opp_get_opp_table(dev)` or first OPP add; it is refcounted and tied to device lifetime via `_opp_table_kref_release`.

REQ-2: OPPs are stored sorted ascending by primary key (freq, level, or bandwidth depending on `key_type`); `find_freq_ceil` / `_floor` perform a single ordered traversal.

REQ-3: `dev_pm_opp_set_rate(dev, target_freq)` does in order: (a) `_find_freq_ceil`, (b) for each supply, `_set_opp_voltage` (ramp pre-clk if scaling up, post-clk if scaling down), (c) `_set_required_opps` for genpd parents, (d) `config_clks` callback (or `dev_pm_opp_config_clks_simple`), (e) `_set_opp_bw` for interconnect, (f) `_set_opp_level` for arbitrary level if requested.

REQ-4: For multi-supply devices, supply order in `opp->supplies[]` matches `opp-microvolt-<name>` declaration order; voltage ramp serializes supplies but does not require all to be programmed atomically.

REQ-5: `supported-hw` mask: an OPP is "supported" iff (`opp_table->supported_hw[i] & opp->supported_hw[i]) != 0` for every index; unsupported OPPs are filtered from lookups but kept in the table.

REQ-6: `required-opps` chain: each OPP can name one or more peer OPPs in other tables (e.g. CPU OPP → genpd performance state); chain depth bounded by `opp_table->required_opp_count`; `_set_required_opps` propagates in topological order.

REQ-7: `dev_pm_opp_of_add_table` (REQ for OF-bound clients): parse the `operating-points-v2` node referenced by `dev->of_node`; lazy-link `required-opps` references when the target table is not yet present (`lazy_opp_tables` list).

REQ-8: cpufreq glue (`cpu.c`): `dev_pm_opp_init_cpufreq_table` produces a `cpufreq_frequency_table` snapshot; `dev_pm_opp_of_cpumask_add_table` installs the same OPP table against every CPU in the cpumask.

REQ-9: `dev_pm_opp_register_notifier` exposes per-table notifier callbacks fired on OPP add/remove/enable/disable/voltage-adjust/rate-change.

REQ-10: Debugfs (`debugfs.c`) under `/sys/kernel/debug/opp/<dev>/` lists per-OPP rate, voltage, level, bandwidth, required-opps, and enable bit when `CONFIG_DEBUG_FS` is set.

REQ-11: Energy Model registration (`dev_pm_opp_of_register_em`) requires either `opp-microwatt` (preferred) or `dynamic-power-coefficient` + `opp-microvolt` to derive power; missing data → EM registration declined, not a hard failure.

## Acceptance Criteria

- [ ] AC-1: On a multi-OPP ARM SoC, `cat /sys/kernel/debug/opp/<cpu>/opp_table` lists every devicetree-declared OPP with correct freq/voltage/level.
- [ ] AC-2: `cpupower frequency-set -f <X>MHz` ultimately calls `dev_pm_opp_set_rate(dev, X*1000)` and the regulator reports the matching voltage on `regulator_get_voltage`.
- [ ] AC-3: Scaling down voltage post-clk is observable in trace_clock-tracepoints; ramping up pre-clk is observable on rise transitions.
- [ ] AC-4: Adjusting voltage at runtime via `dev_pm_opp_adjust_voltage` reflects on next `set_rate`; notifiers fire with `OPP_EVENT_ADJUST_VOLTAGE`.
- [ ] AC-5: `dev_pm_opp_of_register_em` produces a valid `em_perf_domain` visible in `/sys/devices/system/cpu/cpu0/cpufreq/energy_performance_*`.
- [ ] AC-6: `required-opps` correctly cause genpd performance state to follow CPU OPP changes (observe `genpd_set_performance_state` calls).
- [ ] AC-7: Removing an OPP table on module unload frees all OPPs, regulators, interconnect handles, and genpd votes (kmemleak clean).
- [ ] AC-8: kselftest `tools/testing/selftests/cpufreq/` OPP-driven cases pass on a representative SoC.

## Architecture

`OppTable` in `drivers::opp::OppTable`:

```
struct OppTable {
  refcount: KRef,
  dev_list: Mutex<Vec<Arc<OppDevice>>>,         // multiple devices may share a table
  opp_list: Mutex<Vec<Arc<Opp>>>,               // sorted ascending by key
  clk_count: u32,
  clks: KBox<[Clk]>,
  config_clks: Option<ConfigClksCb>,
  config_regulators: Option<ConfigRegulatorsCb>,
  regulator_count: u32,
  regulators: KBox<[Regulator]>,
  required_opp_count: u32,
  required_opp_tables: KBox<[Arc<OppTable>]>,
  supported_hw: KBox<[u32]>,
  paths: KBox<[IccPath]>,                        // interconnect paths
  current_opp: Mutex<Option<Arc<Opp>>>,
  enabled: bool,
  rate_clk_single: u64,                          // cached for single-clk devices
  key_type: OppKeyType,                          // FREQ / LEVEL / BANDWIDTH
}

struct Opp {
  refcount: KRef,
  available: bool,
  dynamic: bool,
  turbo: bool,
  pstate: u32,
  level: u32,
  rates: KBox<[u64]>,                            // per-clk
  bandwidth: KBox<[OppBw]>,                      // per-path
  supplies: KBox<[OppSupply]>,                   // per-regulator
  required_opps: KBox<[Arc<Opp>]>,
  supported_hw: KBox<[u32]>,
  np: Option<DeviceNode>,
}
```

`OppTable::set_rate(dev, target_freq)` (the main hot path):
1. `opp_table = _find_opp_table(dev)` (lazily allocate if absent).
2. `target_opp = _find_freq_ceil(opp_table, &target_freq)`.
3. Determine scaling direction: `scaling_up = target_freq > current_freq`.
4. If scaling up: `_set_opp_voltage(dev, supplies[i], target_opp->supplies[i])` for each supply (raise voltage first).
5. `_set_required_opps(dev, opp_table, target_opp, scaling_up)`: propagate to genpd parents in topological order.
6. `config_clks(dev, opp_table, opp, /*scaling_down=*/!scaling_up)` (or `dev_pm_opp_config_clks_simple` for single-clk default).
7. If scaling down: `_set_opp_voltage(...)` for each supply (lower voltage after clk).
8. `_set_opp_bw(opp_table, target_opp, dev)`: program interconnect path bandwidth.
9. `_set_opp_level(dev, target_opp)`: if level-bound, write level.
10. Update `opp_table->current_opp`.

`OfBinding::init_opp_table` (`_of_init_opp_table`):
1. Read `operating-points-v2` node; iterate child OPP nodes.
2. For each child: parse `opp-hz`, `opp-microvolt[-<name>]`, `opp-microamp[-<name>]`, `opp-level`, `opp-peak-kBps`, `opp-supported-hw`, `required-opps`, `opp-suspend`.
3. `_opp_add` inserts in sorted order; duplicate (per `_opp_is_duplicate`) is rejected with `-EEXIST`.
4. Required-opps phandles are resolved lazily through `lazy_opp_tables` if the target table is not yet registered.

CPU glue `CpuBinding::init_cpufreq_table` (`cpu.c:dev_pm_opp_init_cpufreq_table`):
1. Iterate `opp_table->opp_list`; produce one `cpufreq_frequency_table` entry per `available && rates[0] != 0` OPP.
2. Skip unsupported (via `supported_hw` mask) entries; sentinel terminator `CPUFREQ_TABLE_END`.
3. Caller (cpufreq driver) keeps a handle for `cpufreq_frequency_table_target` lookups.

Sharing-mask discovery `OfBinding::get_sharing_cpus`:
1. From `dev->of_node`, find the `opp-shared` flag on the OPP table node.
2. If set: every CPU whose DT `operating-points-v2` phandle equals this one is added to the sharing cpumask.
3. Used by cpufreq policy to discover sibling CPUs sharing the same OPP table (cluster-wide cpufreq).

## Hardening

- All OPP add/remove/adjust APIs require either a kernel client (driver) or, when exposed via debugfs writeback, `CAP_SYS_ADMIN`; userspace cannot mint phantom OPPs.
- `_opp_compare_key` / `_opp_is_duplicate` enforce strict ordering so no two OPPs share the same primary key — defeats lookup ambiguity and accidental UAF on the result.
- `_set_opp_voltage` validates `u_volt_min ≤ u_volt ≤ u_volt_max` before calling `regulator_set_voltage`; refuses to drop below the regulator's machine constraints.
- `_set_opp_bw` validates `peak ≤ icc_path->avg_max` to refuse interconnect overcommit.
- `required_opps` chain depth is bounded by `required_opp_count` at table-init time; cycles are rejected (a required-opp graph must be DAG).
- `_remove_opp_table` waits for in-flight `set_rate` calls via mutex, then walks `dev_list` to detach every device before freeing.
- Devicetree OPP parser validates `opp-microvolt` array length against `supplies_count` and rejects malformed counts.
- Lazy required-opp linking holds a mutex over `lazy_opp_tables`; if a lazy link cannot resolve at probe completion, `_required_opps_available` marks the dependent OPPs unsupported.
- Refcount paths use `kref_get_unless_zero` on the dereference side so a concurrent `dev_pm_opp_remove_all_dynamic` cannot UAF an OPP being looked up.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `opp_table`, `dev_pm_opp`, `opp_device`, supply/bandwidth/required-opp arrays; debugfs reads cross only `seq_file` helpers.
- **PAX_KERNEXEC** — `dev_pm_opp_*` exported functions in W^X regions; `config_clks` / `config_regulators` callback tables in `__ro_after_init` once `dev_pm_opp_set_config` lands.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `dev_pm_opp_set_rate`, `_set_required_opps`, and `_of_add_opp_table_v2` to break leaks of regulator-table-base pointers.
- **PAX_REFCOUNT** — saturating `refcount_t` on `opp_table->refcount` and `opp->refcount`; overflow trap defeats double-`put_opp_table` and required-opps-cycle UAF.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `opp_table`, `dev_pm_opp`, supply arrays, required-opp arrays, and interconnect path handles so stale OPP keys cannot bleed across module reload.
- **PAX_UDEREF** — SMAP/PAN enforced on every debugfs writer (`OppTable` debugfs writeback) and on `dev_pm_opp_add_dynamic` from sysfs adapters; reject user pointers outside canonical helpers.
- **PAX_RAP / kCFI** — `dev_pm_opp_data` callback tables, `config_clks_simple`, `_opp_config_regulator_single`, and registered notifier blocks marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `opp_table`/`dev_pm_opp` pointers and regulator handles behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict OPP-add-fail, regulator-set-fail, and `_set_required_opps` recursion banners to CAP_SYSLOG so attackers cannot probe SoC voltage-rail layout via dmesg.
- **opp-table PAX_REFCOUNT** — every `dev_pm_opp_get_opp_table_ref` saturates on overflow; debugger-driven add/remove storms cannot wrap the refcount.
- **of-parse range checks** — `opp-microvolt[-<name>]` array length, `opp-hz` u64 bound, `opp-level` u32 bound, and `opp-supported-hw` array length all validated against `supported_hw_count` before any allocation; refuse malformed nodes.
- **Voltage regulator-set bound** — `_set_opp_voltage` enforces `u_volt_min ≤ u_volt ≤ u_volt_max` and refuses to set a voltage outside the regulator's machine-defined constraint; defeats DT-driven OPP that would brown out or smoke a rail.
- **Debugfs CAP_SYS_ADMIN** — `/sys/kernel/debug/opp/` writers (enable/disable, adjust_voltage) gated by CAP_SYS_ADMIN against the owning user namespace; read-only attributes inherit the kernel debugfs default mode.
- **Required-opps DAG check** — `_required_opps_available` rejects cycles; depth bounded by `required_opp_count` to prevent unbounded recursion during `_set_required_opps`.

Rationale: an OPP table is the kernel's contract about which (frequency, voltage, bandwidth) tuples are safe to apply to a SoC. A duplicate OPP, a malformed `opp-microvolt` array, an out-of-bound voltage write, or a required-opps cycle would either brown out a rail, spin a worker forever, or UAF an `Arc<Opp>` mid-`set_rate`. RAP/kCFI on `config_clks`/`config_regulators` vtables, CAP_SYS_ADMIN on debugfs writes, refcount-overflow trapping, DT array-length validation, regulator min/max bounds, and DAG-checked required-opps turn the OPP framework from "DT says we boost to 2.4 GHz" into a structurally-validated, regulator-safe, hotplug-clean performance-state engine.

## Open Questions

- The lazy required-opp link path (`lazy_link_required_opps`) currently re-walks `lazy_opp_tables` on every new table registration; this is O(N²) in deeply nested SoC topologies. Tier-3 documents the behaviour; a refactor to a graph-keyed map is out of scope here.
- Energy Model registration semantics when `opp-microwatt` is absent but `dynamic-power-coefficient` is present should be unified across `cpufreq-dt` and `devfreq` clients — open for follow-up.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `opp_list_sorted` | ORDERING | `_opp_add` maintains ascending sort by primary key. |
| `required_opps_dag` | INVARIANT | `_opp_set_required_opps` rejects cycles; chain depth ≤ `required_opp_count`. |
| `supplies_bounded` | OOB | `opp->supplies[]` length equals `opp_table->regulator_count`. |
| `opp_kref_no_uaf` | UAF | `dev_pm_opp_put` decrements refcount; `_opp_kref_release` runs only on zero. |

### Layer 2: TLA+

`models/opp/set_rate.tla`: prove the regulator-vs-clk ordering protocol (raise volts → bump clk on scaling up; bump clk → lower volts on scaling down) is observed under concurrent `dev_pm_opp_set_rate` calls from cpufreq and devfreq on the same device.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `OppTable::set_rate` post: `opp_table.current_opp == target_opp` AND every regulator reads back `target_opp.supplies[i].u_volt` | `OppTable::set_rate` |
| `OfBinding::init_opp_table` post: every parsed OPP is sorted into `opp_list` and indexed by primary key | `OfBinding::init_opp_table` |
| `_set_required_opps` post: every required-opp in the chain has its target performance state applied | `OppTable::set_required_opps` |

### Layer 4: Verus/Creusot functional

cpufreq governor demands frequency F → `dev_pm_opp_set_rate(dev, F)` → first matching ceil OPP O → regulators set to O.supplies → clk programmed to O.rates → required-OPPs applied → bandwidth voted. Encoded as a chain of Verus invariants where each step's postcondition is the next step's precondition.
