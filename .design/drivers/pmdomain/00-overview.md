# Tier-3: drivers/pmdomain/ — generic PM domain (genpd) subsystem overview

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pmdomain/00-overview.md
upstream-paths:
  - drivers/pmdomain/core.c
  - drivers/pmdomain/governor.c
  - include/linux/pm_domain.h
-->

## Summary

`drivers/pmdomain/` (formerly `drivers/base/power/domain*.c`) implements the *generic PM domain* (genpd) framework: a kernel abstraction over SoC power domains (groups of peripherals that share a single power-rail / clock-gate / reset toggle). A genpd consumer attaches a device to one or more genpds via DT `power-domains` / ACPI `_PSD`, and the genpd core takes over runtime-PM: when every device in the domain is runtime-suspended (and every child subdomain is off), the core can call the backend `power_off(genpd)` callback and remove power; conversely, the first runtime-resume in a domain triggers `power_on(genpd)`. Optionally, a *governor* (`governor.c`) decides which of several `genpd_power_state[]` idle states to enter, based on the next predicted wakeup, residency, and exit latency.

This Tier-3 covers the subsystem shape: `struct generic_pm_domain` (per-domain control block), `struct dev_pm_domain` (the per-device PM-domain ops table that genpd installs), governors (`simple_qos_governor`, `pm_domain_always_on_gov`, `pm_domain_cpu_gov`), DT providers via `of_genpd_add_provider_*` (in `drivers/pmdomain/core.c`), per-vendor pmdomain drivers under `drivers/pmdomain/{actions,amlogic,apple,arm,bcm,imx,marvell,mediatek,qcom,renesas,rockchip,samsung,st,starfive,sunxi,tegra,thead,ti,xilinx}/` (each is a `platform_driver` that registers one or more `generic_pm_domain` instances and provides `power_on`/`power_off` callbacks against vendor-specific MMIO).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct generic_pm_domain` | per-domain: dev, domain (dev_pm_domain), parent_links, child_links, dev_list, gov, gd, name, sd_count, status, device_count, performance_state, cpus, power_off, power_on, attach_dev, detach_dev, set_performance_state, set_hwmode_dev, flags, states[], state_count, state_idx, lock_ops | `drivers::pmdomain::Domain` |
| `struct gpd_link` | parent↔child link with per-link performance_state | `drivers::pmdomain::Link` |
| `struct generic_pm_domain_data` | per-attached-device: base (pm_domain_data), td, nb (qos notifier), power_nb, cpu, performance_state, default_pstate, rpm_pstate, opp_token, hw_mode, rpm_always_on | `drivers::pmdomain::PmDomainData` |
| `struct genpd_power_state` | per-state: name, power_off_latency_ns, power_on_latency_ns, residency_ns, usage, rejected, fwnode, idle_time | `drivers::pmdomain::PowerState` |
| `struct dev_power_governor` | system_power_down_ok / power_down_ok / suspend_ok callbacks | `drivers::pmdomain::Governor` |
| `struct genpd_governor_data` | per-domain governor scratchpad: max_off_time_ns, next_wakeup, next_hrtimer, last_enter, cached_power_down_ok | `drivers::pmdomain::GovernorData` |
| `pm_genpd_init(genpd, gov, is_off)` / `pm_genpd_remove(genpd)` | per-domain register / unregister | `Domain::init` / `_remove` |
| `pm_genpd_add_device(genpd, dev)` / `pm_genpd_remove_device(dev)` | attach / detach a device | `Domain::add_device` / `_remove_device` |
| `pm_genpd_add_subdomain(genpd, subdomain)` / `pm_genpd_remove_subdomain` | tree-of-domains relationship | `Domain::add_subdomain` |
| `dev_pm_domain_attach(dev, flags)` / `dev_pm_domain_detach(dev, power_off)` | high-level wrapper used by platform-bus probe (cross-ref `base/platform-device.md`) | `Domain::dev_pm_domain_attach` |
| `dev_pm_domain_attach_list(dev, data, &list)` | attach to multiple PM-domains by name | `Domain::attach_list` |
| `of_genpd_add_provider_simple(np, genpd)` / `_onecell(np, data)` | DT provider register | `Domain::of_add_provider` |
| `simple_qos_governor` / `pm_domain_always_on_gov` / `pm_domain_cpu_gov` | upstream governors in `governor.c` | `Governor::QOS` / `_ALWAYS_ON` / `_CPU` |
| `dev_pm_genpd_set_performance_state(dev, state)` | per-device aggregate performance-state request | `Domain::set_performance_state` |
| `dev_pm_genpd_set_next_wakeup(dev, time)` / `_get_next_hrtimer` | hint next wakeup for governor | `Domain::set_next_wakeup` |
| `dev_pm_genpd_synced_poweroff(dev)` | mark consumer needs synced power-off | `Domain::synced_poweroff` |
| `dev_pm_genpd_set_hwmode(dev, enable)` / `_get_hwmode` | per-device hw-mode (devices that can self-manage power) | `Domain::set_hwmode` |
| `dev_pm_genpd_rpm_always_on(dev, on)` | mark device must keep its domain on for runtime-PM | `Domain::rpm_always_on` |
| `pm_genpd_inc_rejected(genpd, state_idx)` | per-state rejected counter | `Domain::inc_rejected` |

## Compatibility contract

REQ-1: A genpd backend driver fills `struct generic_pm_domain` (name + power_on + power_off + flags + states[]), calls `pm_genpd_init(genpd, gov, is_off)`, then registers as a DT provider via `of_genpd_add_provider_simple` / `_onecell` (or ACPI provider via `dev_pm_domain_attach`).

REQ-2: `pm_genpd_init` registers `genpd` in `gpd_list`, attaches its `dev_pm_domain` ops (override of runtime-PM and system-PM), and assigns `genpd->lock_ops` based on `GENPD_FLAG_IRQ_SAFE` (raw spinlock vs. mutex).

REQ-3: Device attach (`pm_genpd_add_device` or `dev_pm_domain_attach`): allocates `generic_pm_domain_data`, links it into `genpd->dev_list`, increments `genpd->device_count`, installs `genpd` into `dev->pm_domain`.

REQ-4: Runtime-PM integration: genpd installs its own `runtime_suspend` / `runtime_resume` in `dev->pm_domain`; on the *last* device-suspend in the domain, `genpd_power_off` is invoked; on the *first* device-resume, `genpd_power_on` runs.

REQ-5: Subdomain tree: `pm_genpd_add_subdomain(parent, child)` links them via `gpd_link`; a parent cannot power off while any child is on (`sd_count > 0`); a child cannot power on without first powering its parent.

REQ-6: Multi-state idle: `genpd->states[]` describes N idle states (sorted shallowest → deepest); per-state `power_off_latency_ns` and `residency_ns` measured and refined on each power-off cycle; `state_idx` chosen by governor.

REQ-7: Performance states (a.k.a. OPP / DVFS for power domains): `dev_pm_genpd_set_performance_state(dev, state)` aggregates per-device requests via max(); `_genpd_set_performance_state(genpd, max)` calls backend `set_performance_state(genpd, state)`.

REQ-8: Notifier chain: consumers register on `genpd->power_notifiers` for `GENPD_NOTIFY_PRE_OFF` / `GENPD_NOTIFY_OFF` / `GENPD_NOTIFY_PRE_ON` / `GENPD_NOTIFY_ON` events; robust call chain rewinds on PRE-failure.

REQ-9: System suspend/resume integration: `genpd_prepare` / `_complete` and `genpd_{suspend,resume,freeze,thaw,poweroff,restore}_noirq` hook into `pm_ops`; on `prepare_count > 0` the domain refuses to power-off during runtime-PM.

REQ-10: CPU domain (`GENPD_FLAG_CPU_DOMAIN`): integrates with `cpuidle` last-man-standing — the last CPU entering idle in the domain triggers `genpd_power_off`; first CPU exiting triggers `_power_on`.

REQ-11: Always-on / RPM-always-on / stay-on flags (`GENPD_FLAG_ALWAYS_ON`, `_RPM_ALWAYS_ON`, `GENPD_FLAG_ACTIVE_WAKEUP`) suppress power-off in different situations (system suspend vs. runtime-PM vs. wakeup-path).

REQ-12: debugfs (`<debugfs>/pm_genpd/`) exposes per-domain status, devices list, idle-state stats, governor choices.

## Acceptance Criteria

- [ ] AC-1: Boot on a Qualcomm / i.MX / Rockchip / Tegra reference board populates `<debugfs>/pm_genpd/` with the expected per-SoC domains (gpu, video, modem, peripheral, ...).
- [ ] AC-2: A peripheral device attached to a genpd correctly powers off the domain after `pm_runtime_put` on the last device.
- [ ] AC-3: Subdomain tree: parent stays on while any child is on; logs power-off-rejection counter on `pm_genpd_inc_rejected`.
- [ ] AC-4: Performance-state aggregation: two devices on the same domain request different OPPs; aggregated max() is what the backend programs.
- [ ] AC-5: System suspend cycle: every always-on domain stays powered; every other domain transitions through its deepest idle state on `S3`/`S0ix`.
- [ ] AC-6: Governor swap (debugfs) requires CAP_SYS_ADMIN and atomically replaces `genpd->gov`.

## Architecture

```
+-----------------------------------+
| backend driver (e.g. imx_pgc.c)   |
|   - power_on / power_off          |
|   - set_performance_state         |
|   - states[]                      |
|   - of_provider                   |
+------------+----------------------+
             |
             v
+-----------------------------------+    +-------------------+
| genpd core (drivers/pmdomain/     |--->| dev_power_governor|
|             core.c)               |    |  (governor.c)     |
|   gpd_list, gpd_list_lock         |    +-------------------+
|   per-genpd lock (mutex/spin/raw) |
|   notifier chain                  |
|   subdomain tree (gpd_link)       |
|   debugfs                         |
+------------+----------------------+
             |
             v
+-----------------------------------+
| dev->pm_domain installed          |
|   runtime_suspend / runtime_resume|
|   suspend_noirq / resume_noirq    |
+-----------------------------------+
```

Backend drivers in `drivers/pmdomain/{vendor}/` are `platform_driver`s that probe per-SoC PMC / SCU / SCMI / RPMH controllers. Each backend constructs N `generic_pm_domain` instances, fills `power_on` / `power_off` to poke MMIO registers (or send mailbox messages to the firmware), populates `states[]` from DT, attaches `simple_qos_governor` or a domain-specific governor, calls `pm_genpd_init` for each, and finally registers a single `of_genpd_provider` covering all of them.

The core arbitrates everything: it owns the global `gpd_list`, walks the subdomain tree for power-off propagation, talks to runtime-PM in `__genpd_runtime_suspend` / `_resume`, drives the governor for state choice, and routes attached devices through `dev->pm_domain` redirection.

## Hardening

- Per-genpd lock ops (mutex / spin / raw-spin) selected via `GENPD_FLAG_IRQ_SAFE` so atomic backends never sleep in atomic context.
- `gpd_list_lock` is mutex; per-genpd structures protected by their own lock.
- Robust notifier chain (`raw_notifier_call_chain_robust`) rewinds on PRE-failure so consumers don't see half-applied transitions.
- `genpd_status_on` / `genpd_is_always_on` checks guard every power-off attempt.
- `prepared_count` blocks runtime-PM power-off while system suspend is in progress.
- Subdomain bookkeeping (`sd_count`) prevents parent power-off while children are on.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `generic_pm_domain`, `gpd_link`, `generic_pm_domain_data`, `genpd_governor_data`, `genpd_power_state[]`, and debugfs string buffers; sysfs prints length-bounded.
- **PAX_KERNEXEC** — `core.c` + `governor.c` text W^X; `simple_qos_governor`, `pm_domain_always_on_gov`, `pm_domain_cpu_gov`, and per-genpd `dev_pm_domain` ops in `__ro_after_init` cells.
- **PAX_RANDKSTACK** — randomize kernel stack offset across `genpd_power_on`, `_power_off`, `_runtime_suspend`, `_runtime_resume`, `_suspend_noirq`, governor entry points.
- **PAX_REFCOUNT** — saturating `refcount_t` on `genpd->device_count`, `genpd->sd_count`, and per-attached-device `generic_pm_domain_data` references; overflow trap defeats attach/detach race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `generic_pm_domain` (on `pm_genpd_remove`), `gpd_link`, `generic_pm_domain_data`, `genpd_governor_data`, and per-state save/restore buffers so stale ownership cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every debugfs / sysfs entry into genpd; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `dev_pm_domain` ops, backend `power_on` / `power_off` / `set_performance_state` / `attach_dev` / `detach_dev`, governor `power_down_ok` / `system_power_down_ok`, and the notifier chain entries kCFI-typed with `__ro_after_init` storage.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of backend `power_on` / `power_off` callbacks and per-genpd register-base pointers behind CAP_SYSLOG; suppress `%p` in genpd tracepoints.
- **GRKERNSEC_DMESG** — restrict genpd power-on/off latency-exceeded banners, rejection logs, and governor traces to CAP_SYSLOG so attackers cannot profile SoC power-state behavior via dmesg.
- **Governor RAP** — `dev_power_governor` vtable `__ro_after_init`; kCFI-typed indirect dispatch; refuses corrupted governor pointer install.
- **Attach/detach refcounted** — `pm_genpd_add_device` / `_remove_device` increments / decrements with saturating refcount; double-attach or detach-while-busy refused with -EBUSY.
- **sysfs gated** — `<sysfs>/devices/.../power/pm_qos_resume_latency_us`, `<debugfs>/pm_genpd/*` writes require CAP_SYS_ADMIN; refuse mode-bit relaxation.
- **Notifier list size bounded** — per-domain `power_notifiers` chain capped; refuse pathological N-deep registration that would defeat the robust-call-chain.
- **Subdomain depth bounded** — `pm_genpd_add_subdomain` enforces a hard limit on tree depth so `genpd_lock_nested` lockdep depth never exceeds platform-supported `MAX_LOCK_DEPTH`.

Rationale: genpd is the kernel's authority on which SoC power rails are on at any moment, directly impacting wake-from-suspend latency, secure-boot rollback paths (some SoCs gate eFuse access on a specific domain being off), and side-channel surface (peripherals retain MMIO content while powered). A corrupted `power_on` callback, a missed CAP_SYS_ADMIN check on governor swap, a refcount underflow on `device_count`, or an unbounded subdomain tree can silently keep secure-only rails on, defeat S3 wake, or panic with lockdep depth overflow. Refcount saturation, kCFI on governor + backend dispatch, CAP_SYS_ADMIN on sysfs/debugfs, and subdomain-depth bounds turn genpd into a structural power-policy boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `device_count_no_overflow` | OVERFLOW | `genpd->device_count` saturates on add; refuses remove below zero. |
| `sd_count_no_overflow` | OVERFLOW | `genpd->sd_count` saturates; refuses negative. |
| `state_idx_no_oob` | OOB | `state_idx < state_count` always; governor choices clamped. |
| `subdomain_no_cycle` | INVARIANT | `pm_genpd_add_subdomain` refuses if it would create a cycle in the gpd_link DAG. |

### Layer 2: TLA+

`models/pmdomain/parent_child.tla`: proves that under arbitrary interleavings, a parent genpd is powered on whenever any child is on; concurrent `add_subdomain` and `power_off` admit only consistent observations.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `pm_genpd_add_device` post: `dev->pm_domain == &genpd->domain` AND device in `genpd->dev_list` | `Domain::add_device` |
| `genpd_power_off` post: `genpd->status == GENPD_STATE_OFF` AND every parent `sd_count` decremented | `Domain::power_off` |
| `dev_pm_genpd_set_performance_state` post: aggregated `genpd->performance_state == max(per-device requests)` | `Domain::set_performance_state` |

### Layer 4: Verus/Creusot functional

`pm_genpd_init → add_device(x N) → pm_runtime_get_sync(each) → ... → pm_runtime_put(each) → power_off → remove_device(x N) → pm_genpd_remove` leaves every refcount at zero, no dangling `dev_pm_domain`, no leaked `generic_pm_domain_data`.

## Out of Scope

- Per-vendor pmdomain backends (covered in future `drivers/pmdomain/{vendor}/*.md`)
- SCMI / SCPI / RPMH firmware-mailbox details (covered in `firmware/` docs future)
- Runtime-PM core (`drivers/base/power/runtime.c`) (future `power-runtime.md`)
- System suspend `pm_ops` (future `power-suspend.md`)
- 32-bit-only paths
- Implementation code
