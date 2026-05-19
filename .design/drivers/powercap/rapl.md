# Tier-3: drivers/powercap/intel_rapl_{common,msr,tpmi}.c — Intel RAPL (Running Average Power Limit) powercap zones + per-package PMU + TPMI MMIO interface

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/powercap/intel_rapl_common.c
  - drivers/powercap/intel_rapl_msr.c
  - drivers/powercap/intel_rapl_tpmi.c
  - drivers/powercap/powercap_sys.c
  - include/linux/intel_rapl.h
-->

## Summary

Intel RAPL exposes per-domain energy counters and configurable power-limit constraints (PL1/PL2/PL3/PL4) through the generic powercap sysfs framework. `intel_rapl_common.c` is the topology- and protocol-agnostic core: it discovers RAPL packages (one per socket on server, one per die on client), enumerates per-package domains (PACKAGE, DRAM, PP0/CORE, PP1/UNCORE, PSYS), registers each as a `powercap_zone` with constraint operations, and (optionally) exposes a per-package `perf_event_source` (`rapl_pmu`) so `perf stat -e power/energy-pkg/` can read energy counters as standard perf events. `intel_rapl_msr.c` is the per-CPU MSR backend (legacy interface, e.g. `MSR_PKG_ENERGY_STATUS = 0x611`, `MSR_PKG_POWER_LIMIT = 0x610`, `MSR_DRAM_ENERGY_STATUS = 0x619`). `intel_rapl_tpmi.c` is the per-die TPMI (Topology Aware Register and PM Capsule Interface) MMIO backend used on Sapphire Rapids and later, where the same RAPL register layout is replicated in an auxbus-attached MMIO region accessible without a per-CPU IPI. RAPL energy counters are also one of the best-known kernel side channels (Platypus): per-domain microjoule resolution leaks instruction-level operand power, so reading any energy file from a non-root context must be denied.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rapl_package` | per-socket/per-die state (id, domains[], priv backend, pmu) | `drivers::rapl::Package` |
| `struct rapl_domain` | per-domain (PACKAGE/DRAM/PP0/PP1/PSYS) state + powercap zone | `drivers::rapl::Domain` |
| `struct rapl_if_priv` | backend vtable (read_raw, write_raw, defaults, reg map) | `drivers::rapl::Backend` |
| `rapl_add_package(id, priv, id_is_cpu)` / `rapl_remove_package(rp)` | per-package register/unregister | `Subsystem::add_package` / `_remove_package` |
| `rapl_detect_domains(rp)` | walk `RAPL_DOMAIN_*` enum; register valid ones as powercap zones | `Package::detect_domains` |
| `rapl_package_register_powercap(rp)` | per-package powercap registration | `Package::register_powercap` |
| `zone_ops[]` (`get_energy_uj`, `release_zone`, `set_enable`, `get_enable`, `get_max_energy_range_uj`) | powercap zone vtable | `Domain::zone_ops` |
| `constraint_ops` (`set_power_limit_uw`, `get_power_limit_uw`, `set_time_window_us`, `get_time_window_us`, `get_max_power_uw`, `get_name`) | per-PL constraint vtable | `Domain::constraint_ops` |
| `rapl_read_pl_data(rd, pl, prim, atomic)` / `rapl_write_pl_data(rd, pl, prim, value)` | per-PL primitive accessors | `Domain::read_pl` / `_write_pl` |
| `rapl_unit_xlate(rd, type, value, to_raw)` | apply per-package unit (energy/power/time) | `Domain::unit_xlate` |
| `package_power_limit_irq_save(rp)` / `_irq_restore(rp)` | PL irq-enable bit save/restore | `Package::pl_irq_save` / `_irq_restore` |
| `rapl_msr_read_raw(cpu, ra, pmu_ctx)` / `_write_raw(cpu, ra)` | per-CPU MSR backend | `MsrBackend::read_raw` / `_write_raw` |
| `tpmi_rapl_read_raw(id, ra, atomic)` / `_write_raw(id, ra)` | per-die TPMI MMIO backend | `TpmiBackend::read_raw` / `_write_raw` |
| `parse_one_domain(trp, offset)` (TPMI) | walk MMIO domain headers, populate `regs[]` | `TpmiBackend::parse_one_domain` |
| `rapl_pmu` + `rapl_pmu_event_init/add/del/start/stop/read` (PMU path) | per-package `perf_event_source` | `Package::pmu` |
| `rapl_package_add_pmu(rp)` / `rapl_package_remove_pmu(rp)` | per-package PMU registration | `Package::add_pmu` / `_remove_pmu` |

## Compatibility contract

REQ-1: Each RAPL package registers as a single powercap parent zone under control-type `intel-rapl` (legacy MSR) or `intel-rapl-mmio` / `intel-rapl-tpmi` (MMIO/TPMI), with per-domain child zones named per `rapl_domain_names[]`: `package-N`, `dram`, `core`, `uncore`, `psys`.

REQ-2: Per-domain `energy_uj` is monotonically non-decreasing (32-bit wraparound handled in `rapl_event_update` / `get_energy_counter`); reported in microjoules after `unit_xlate` from MSR/TPMI ESU (Energy Status Units, ~61 µJ on most parts; ~15.3 µJ on Atom DRAM).

REQ-3: Per-domain constraints expose PL1 (long-term, ~28 s window typical), PL2 (short-term, ~2.44 ms), PL3 (peak, instantaneous), PL4 (max-power) where supported by the part. Per-PL files: `constraint_<n>_power_limit_uw`, `_time_window_us`, `_max_power_uw`, `_name`.

REQ-4: `set_power_limit_uw` writes the PL clamp bit and PL value with unit translation; `_time_window_us` writes Y/F-encoded time window per IA32_PKG_POWER_LIMIT format.

REQ-5: PL-IRQ save/restore: `package_power_limit_irq_save` reads and clears the IRQ-on-PL-exceed bit at suspend so wakes do not fire spuriously; restore on resume.

REQ-6: MSR backend (`intel_rapl_msr`): per-CPU MSR access via `rdmsr_safe_on_cpu` / `wrmsr_safe_on_cpu` (or `smp_call_function_single` for PMU-atomic contexts); CPU hotplug callbacks `rapl_cpu_online` / `_down_prep` handle migration of the package-cpumask.

REQ-7: TPMI backend (`intel_rapl_tpmi`): binds as an auxiliary driver against device id `intel_vsec.tpmi-rapl`; iomap'd MMIO region, per-domain header at offset, `parse_one_domain` discovers RAPL_DOMAIN_PACKAGE/DRAM/PP0/PP1/PSYS layouts.

REQ-8: PMU path (`CONFIG_PERF_EVENTS=y`): per-package `pmu` instance with `cpumask` of one CPU per package; events `power/energy-pkg/`, `energy-dram`, `energy-cores`, `energy-gpu`, `energy-psys`; `event_read_counter` translates ESU → joules-per-period via `event->hw.last_period`.

REQ-9: Per-CPU `x86_cpu_id` table (`rapl_ids[]`) selects the appropriate `rapl_defaults` (MSR offsets, PL count, unit register, etc.) per Intel/AMD model; `rapl_defaults_amd` for AMD F17h+.

REQ-10: Default-enabled state per `rapl_msr_priv->reg_unit` presence; absent unit register → refuse to register domain.

REQ-11: On unbind (module unload or auxbus detach), unregister all child zones first, then the parent package, then free the `rapl_package`.

## Acceptance Criteria

- [ ] AC-1: On an Intel Coffee Lake desktop, `ls /sys/class/powercap/` shows `intel-rapl:0`, `intel-rapl:0:0` (core), `intel-rapl:0:1` (uncore), `intel-rapl:0:2` (dram, if present).
- [ ] AC-2: `cat /sys/class/powercap/intel-rapl:0/energy_uj` increases monotonically under load; rate matches CPU power draw within ±10 %.
- [ ] AC-3: Writing `/sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw = 25000000` (25 W PL1) actually throttles the package under sustained `stress-ng --cpu`.
- [ ] AC-4: PL clamp-bit asserted appears in MSR readback for `MSR_PKG_POWER_LIMIT` (bit 15 / bit 47).
- [ ] AC-5: `perf stat -a -e power/energy-pkg/ sleep 10` reports a non-zero value (PMU path).
- [ ] AC-6: On Sapphire Rapids+ with TPMI, `ls /sys/class/powercap/intel-rapl-tpmi:0` also enumerates domains; values track the MSR backend within ±2 %.
- [ ] AC-7: Suspend/resume preserves PL state; `dmesg` shows `package_power_limit_irq_save`/`_restore` invocations.
- [ ] AC-8: kselftest `tools/testing/selftests/powercap/` energy-counter monotonicity case passes.
- [ ] AC-9: Side-channel posture: opening `/sys/class/powercap/intel-rapl:0/energy_uj` as a non-root user returns `-EACCES` once Platypus-mitigation gate is applied.

## Architecture

`Package` in `drivers::rapl::Package`:

```
struct Package {
  id: i32,                              // physical package id (or die id)
  domains: [Option<Domain>; RAPL_DOMAIN_MAX],
  priv_: Arc<Backend>,                  // MSR / MMIO / TPMI vtable
  cpumask: Cpumask,                     // CPUs in this package
  power_zone: PowercapZone,             // parent zone
  pmu: Option<KBox<RaplPmu>>,
  pl_irq_state: u64,                    // saved IRQ-on-PL bits across suspend
}

struct Domain {
  id: RaplDomainKind,
  name: KStr,
  power_zone: PowercapZone,             // /sys/class/powercap/intel-rapl:N:M/
  rpl: [PowerLimit; NR_POWER_LIMITS],   // PL1..PL4
  regs: [Register; RAPL_DOMAIN_REG_MAX],
  domain_energy_unit: u32,              // microjoules per LSB
  ulong_max_energy: u64,                // wrap point for monotonic accounting
}
```

Per-package init `Package::register_powercap`:
1. `rapl_detect_domains(rp)`: iterate `RAPL_DOMAIN_PACKAGE`, `RAPL_DOMAIN_PP0`, `RAPL_DOMAIN_PP1`, `RAPL_DOMAIN_DRAM`, `RAPL_DOMAIN_PLATFORM`; for each, probe `priv->regs[domain]` and `rapl_check_domain` (read STATUS register; absent → skip).
2. For each present domain: register child `powercap_zone` with `zone_ops` + `constraint_ops`.
3. `find_nr_power_limit(rd)` per domain determines PL count.
4. If `CONFIG_PERF_EVENTS`: `rapl_package_add_pmu(rp)` registers the per-package PMU with `cpumask` = one CPU per package.

Per-domain energy read `Domain::get_energy_counter`:
1. `rapl_read_pl_data(rd, 0, ENERGY_STATUS, /*atomic=*/false)`.
2. Result is raw ENERGY_STATUS register × `domain_energy_unit` → microjoules.
3. Handle 32-bit wraparound by maintaining a 64-bit accumulator updated on each read (rate-limited by `rapl_update_domain_data`).

Per-PL constraint write `Domain::set_power_limit`:
1. `contraint_to_pl(rd, cid)` → PL index.
2. `rapl_unit_xlate(rd, POWER_UNIT, value, to_raw=true)` → raw clamp value.
3. `rapl_write_pl_data(rd, pl, POWER_LIMIT, raw)` → write CLAMP/LIMIT/ENABLE bits in MSR or MMIO register.
4. If the part requires "lock" semantics, refuse writes after the lock bit is set.

MSR backend `MsrBackend::read_raw`:
1. If atomic context (PMU read): `rdmsrl_safe` directly on the calling CPU after asserting CPU is in target package.
2. Otherwise: `rdmsr_safe_on_cpu(cpu, msr, &lo, &hi)`.

TPMI backend `TpmiBackend::probe(auxdev)`:
1. `intel_tpmi_resource_count(auxdev)` gives per-die resource count.
2. For each die: `intel_tpmi_get_resource_at_index(auxdev, i)` → `struct resource` → `devm_ioremap_resource`.
3. `parse_one_domain` walks per-domain header bytes (id, offset, size); populate `regs[]` with MMIO pointers.
4. `rapl_add_package(die_id, priv, /*id_is_cpu=*/false)`.

PMU path `Package::add_pmu`:
1. Construct a `struct pmu` with `cpumask` containing one CPU per active package.
2. `event_init` validates `event->attr.config` against `RAPL_PMU_EVENT_*` mask; only known domain codes accepted.
3. `event_start/stop/read` use `rapl_read_pl_data(rd, 0, ENERGY_STATUS, true)` (atomic path) to maintain `event->hw.prev_count`.

## Hardening

- All MSR writes that change PL values, clamp bits, or lock bits require `CAP_SYS_ADMIN`; per-zone sysfs constraint files inherit the powercap-core `CAP_SYS_ADMIN` write gate.
- Energy counter reads default to root-only mode 0400 (Platypus mitigation): the powercap-core sysfs binding is patched so `energy_uj` and PMU `energy-*` events are gated by CAP_SYS_ADMIN; rate-limit periodic polling of the same domain via `rapl_update_domain_data`.
- TPMI register access is bounded by the `devm_ioremap_resource` size returned by `intel_tpmi_get_resource_at_index`; out-of-bounds offsets in `parse_one_domain` are rejected.
- PL value writes are validated against `get_max_power_uw(rd)` from THERMAL_SPEC_POWER (TDP register) — userspace cannot write nonsensical wattages.
- PL time-window writes validate Y/F encoding bounds; reject zero or saturated values.
- Per-CPU MSR access uses `_safe` variants — a missing or locked MSR returns `-EIO` rather than `#GP`-faulting in kernel.
- CPU hotplug callbacks tolerate concurrent unbind: `rapl_remove_package` waits for in-flight `read_raw` to drain.
- The PMU is registered with one event per (package, domain) — event count is bounded by `NR_RAPL_DOMAINS × NR_PACKAGES`, preventing event-table inflation attacks.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `rapl_package`, `rapl_domain`, `rapl_pmu`, per-PMU per-package data, and the `reg_action` staging used by both MSR and TPMI backends.
- **PAX_KERNEXEC** — `zone_ops`, `constraint_ops`, `rapl_pmu` callbacks, MSR/TPMI backend vtables in W^X regions; populated then `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across powercap sysfs writes, `rapl_pmu_event_*`, `rapl_msr_update_func`, and TPMI MMIO accessors.
- **PAX_REFCOUNT** — saturating `refcount_t` on `rapl_package`, `rapl_domain`, and PMU registration handles; overflow trap defeats hotplug/unbind UAF.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `rapl_package`, `rapl_domain`, `reg_action` buffers, and TPMI MMIO descriptor tables so PL state and energy snapshots cannot bleed across module reload.
- **PAX_UDEREF** — SMAP/PAN enforced on every powercap sysfs writer; constraint inputs (`set_power_limit_uw`, `set_time_window_us`) cross only canonical kstrtoull helpers.
- **PAX_RAP / kCFI** — `zone_ops`, `constraint_ops`, `rapl_pmu_*`, `rapl_if_priv->read_raw`/`write_raw`, and the MSR/TPMI backend vtables marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `rapl_packages`, per-package `priv` pointers, and TPMI MMIO bases behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict RAPL-init, PL-overflow, PL-IRQ, and TPMI-register-fail banners to CAP_SYSLOG so attackers cannot map energy-domain layout via dmesg.
- **MSR write CAP_SYS_ADMIN** — every `_write_raw` path (MSR, MMIO, TPMI) goes through a powercap sysfs writer that asserts CAP_SYS_ADMIN against the owning user namespace before dispatching; PL clamps cannot be set from a sandbox.
- **Constraint POWER_LIMIT PAX_USERCOPY** — userspace strings parsed via kstrtoull on whitelisted stack buffers; raw values bounded by `get_max_power_uw` before being written to MSR/MMIO.
- **Energy counter rate-limit (RAPL side channel — Platypus)** — `energy_uj` and PMU `energy-*` reads gated by CAP_SYS_ADMIN by default; per-domain read interval enforced (≥ 1 ms) to defeat instruction-level operand-power leakage. `GRKERNSEC_HIDESYM` extended to suppress `%p` of RAPL counter pointers from non-root.
- **TPMI register bounds** — every MMIO offset checked against `devm_ioremap_resource` size; `parse_one_domain` refuses to register a domain whose header points past the iomap range.
- **PL-IRQ save/restore audit** — PL-IRQ bits saved on suspend, restored on resume; mismatch triggers a refusal to re-enable PL writes until userspace re-asserts.

Rationale: RAPL is simultaneously a power-management control surface (PL writes can starve the platform of frequency budget) and an industry-grade microarchitectural side channel (Platypus extracts AES-NI key bytes from `energy_uj`). RAP/kCFI on `zone_ops`/`constraint_ops`/backend vtables, CAP_SYS_ADMIN on every PL write, CAP_SYS_ADMIN on every energy read, rate-limited per-domain polling, TPMI MMIO bounds-checking, and refusing to re-enable PL-IRQ post-resume on mismatch turn RAPL from "anyone with sysfs can throttle the box and read AES round keys" into a privileged, rate-limited, hotplug-safe constraint surface.

## Open Questions

- Per-zone `energy_uj` mode should be 0400 by default once Platypus mitigation lands platform-wide; coordinate with `powercap_sys.c` so the gate is applied uniformly (not just for intel-rapl).
- TPMI write atomicity across SoC-level constraints (PSYS) wants a per-package mutex shared with MSR backend when both interfaces are live on the same part; out-of-scope for this Tier-3.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pl_value_in_range` | OOB | `set_power_limit_uw` clamps value to [0, `get_max_power_uw`]; never writes a raw clamp larger than the part-defined max. |
| `tpmi_mmio_no_oob` | OOB | every TPMI register access bounded by `devm_ioremap_resource` size. |
| `energy_counter_no_underflow` | INVARIANT | 64-bit accumulator never decreases across 32-bit ENERGY_STATUS wraparounds. |
| `pmu_event_no_uaf` | UAF | `rapl_pmu_event_del` removes event from per-PMU list before `Arc<Package>` ref drop. |

### Layer 2: TLA+

`models/rapl/pl_irq.tla`: prove PL-IRQ save/restore round-trips across S3/S0ix without losing or duplicating constraint state.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `Domain::set_power_limit` post: subsequent `get_current_power_limit` returns the same value (modulo unit truncation) | `Domain::set_power_limit` |
| `Package::register_powercap` post: every present domain has a child zone with a unique `<id>:<idx>` path | `Package::register_powercap` |
| `MsrBackend::read_raw` post: returned energy value is within [prev, prev + max_growth] for the sampling interval | `MsrBackend::read_raw` |

### Layer 4: Verus/Creusot functional

Sysfs writer → `set_power_limit_uw(value)` → `rapl_unit_xlate` → `rapl_write_pl_data` → MSR/MMIO write → readback via `get_current_power_limit` returns the same wattage. Encoded as a Verus invariant tying the userspace input through `unit_xlate` round-tripping.
