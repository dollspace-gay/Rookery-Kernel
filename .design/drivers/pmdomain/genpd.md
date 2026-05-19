# Tier-3: drivers/pmdomain/core.c — genpd core (state machine, governor, attach, runtime-PM)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pmdomain/00-overview.md
upstream-paths:
  - drivers/pmdomain/core.c
  - drivers/pmdomain/governor.c
-->

## Summary

`drivers/pmdomain/core.c` (~4000 LOC) is the engine of the genpd subsystem: it owns the per-domain state machine (`GENPD_STATE_ON` ↔ `GENPD_STATE_OFF`), the attach/detach plumbing that re-routes a device's runtime-PM and system-PM through genpd, the multi-state idle selection (governor + state_idx), the parent↔child subdomain propagation, the performance-state aggregation, the notifier chain (`PRE_OFF` / `OFF` / `PRE_ON` / `ON`), and the system-PM hooks (`prepare`, `suspend_noirq`, `resume_noirq`, `complete`). `governor.c` (~470 LOC) provides three stock governors (`simple_qos_governor`, `pm_domain_always_on_gov`, `pm_domain_cpu_gov`) that consult `next_wakeup` + per-state residency to pick a state index.

This Tier-3 narrows in on the state-machine core: how `genpd_power_on` / `_power_off` validate preconditions and call the backend; how `genpd_runtime_suspend` / `_resume` install themselves as `dev->pm_domain` runtime-PM ops; how `pm_genpd_add_device` / `_remove_device` arbitrate attach lifecycle; how performance-state aggregation works; and how the system-PM `noirq` phases interact with runtime-PM `rpm_pstate` save/restore.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `pm_genpd_init(genpd, gov, is_off)` | per-domain register: alloc gd, init list heads + locks, install `dev_pm_domain` ops, add to gpd_list | `Core::genpd_init` |
| `pm_genpd_remove(genpd)` | unregister: refuse if device_count > 0 or sd_count > 0; drop from gpd_list, free `gd` | `Core::genpd_remove` |
| `pm_genpd_add_device(genpd, dev)` / `_remove_device(dev)` | per-device attach / detach | `Core::add_device` / `_remove_device` |
| `pm_genpd_add_subdomain(genpd, subdomain)` / `_remove_subdomain` | parent↔child link via `gpd_link` | `Core::add_subdomain` |
| `genpd_alloc_dev_data(dev, ...)` / `genpd_free_dev_data(...)` | per-attached-device `generic_pm_domain_data` lifecycle | `Core::alloc_dev_data` / `_free_dev_data` |
| `genpd_power_on(genpd, depth)` / `_power_off(genpd, one_dev_on, depth)` | core state-machine transitions; walk parents on power_on; walk children on power_off | `Core::power_on` / `_power_off` |
| `_genpd_power_on(genpd, timed)` / `_power_off(genpd, timed)` | backend dispatch + latency measurement + notifier chain | `Core::_power_on` / `_power_off` |
| `genpd_runtime_suspend(dev)` / `_runtime_resume(dev)` | installed in `dev->pm_domain.ops.runtime_suspend` / `_resume`; drive subsystem suspend then `genpd_power_off` | `Core::runtime_suspend` / `_resume` |
| `__genpd_runtime_suspend(dev)` / `_resume(dev)` | walk subsystem hierarchy (dev->type / class / bus / driver) for `runtime_suspend` callback | `Core::__runtime_suspend` / `_resume` |
| `genpd_stop_dev(genpd, dev)` / `_start_dev(genpd, dev)` | dispatch `genpd->dev_ops.stop` / `_start` (per-device backend hook) | `Core::stop_dev` / `_start_dev` |
| `genpd_prepare(dev)` / `genpd_complete(dev)` | system suspend prepare/complete: increment `prepared_count` to block runtime-PM power-off | `Core::prepare` / `_complete` |
| `genpd_suspend_noirq(dev)` / `_resume_noirq` / `_freeze_noirq` / `_thaw_noirq` / `_poweroff_noirq` / `_restore_noirq` | per-system-PM-phase noirq hooks | `Core::suspend_noirq` / `_resume_noirq` / ... |
| `genpd_sync_power_off(genpd, ...)` / `_sync_power_on(...)` | unlocked variants for use under `noirq` (system PM already serializes) | `Core::sync_power_off` / `_on` |
| `genpd_switch_state(dev, suspend)` | dev_pm_genpd_suspend / _resume entrypoint used by some subsystem-PM paths | `Core::switch_state` |
| `_genpd_reeval_performance_state(genpd, state)` / `_set_performance_state(genpd, state)` | aggregate + dispatch backend `set_performance_state` | `Core::reeval_perf` / `_set_perf` |
| `_genpd_set_parent_state(genpd, link, parent_state)` / `_rollback_parent_state` | per-parent OPP requirement | `Core::set_parent_state` / `_rollback_parent_state` |
| `genpd_dev_pm_qos_notifier(nb, action, ...)` | dev-PM-QoS constraint notifier — recomputes governor `max_off_time` | `Core::qos_notifier` |
| `genpd_lock_mtx` / `_spin` / `_raw_spin` (and `_nested`, `_interruptible`) | lock-ops dispatch chosen by `GENPD_FLAG_IRQ_SAFE` | `Core::lock_ops` |
| `genpd_debugfs_dir` / `genpd_debug_add` / `_remove` | per-domain debugfs registration (status, devices, idle-state stats) | `Core::debugfs` |
| `simple_qos_governor` (in `governor.c`) | default governor: PM-QoS-driven; chose deepest state whose `power_off_latency + residency < QoS budget` | `Governor::QOS` |
| `pm_domain_always_on_gov` (in `governor.c`) | always returns `false` from `power_down_ok` | `Governor::ALWAYS_ON` |
| `pm_domain_cpu_gov` (in `governor.c`) | CPU-domain governor: integrates with cpuidle last-man-standing | `Governor::CPU` |

## Compatibility contract

REQ-1: `pm_genpd_init(genpd, gov, is_off)` populates `genpd->dev` (allocated via `ida_alloc(&genpd_ida)`), installs `genpd->domain` (`dev_pm_domain`) with `runtime_suspend = genpd_runtime_suspend` / `_resume = _runtime_resume`, picks `lock_ops` based on `GENPD_FLAG_IRQ_SAFE`, sets initial `status = is_off ? OFF : ON`.

REQ-2: `genpd_power_on(genpd, depth)`: under `genpd_lock_nested`, walk `child_links` to power on parents first (recursive with nested-lock depth); call `_genpd_power_on(genpd, timed=true)`; on success `genpd->status = GENPD_STATE_ON`, increment parent `sd_count`.

REQ-3: `genpd_power_off(genpd, one_dev_on, depth)`: refuse if `!genpd_status_on || prepared_count > 0 || GENPD_FLAG_ALWAYS_ON || GENPD_FLAG_RPM_ALWAYS_ON || stay_on || sd_count > 0`. For each child: refuse unless child is in its deepest state. For each attached device: skip if `rpm_always_on`; refuse if any non-IRQ-safe device unsuspended (with `one_dev_on` exception when called from runtime-suspend path). Consult `gov->power_down_ok`. Call `_genpd_power_off`. On success: status = OFF, accounting update, propagate to parents (recursive).

REQ-4: `_genpd_power_on / _off`: dispatch `PRE_ON`/`PRE_OFF` notifier (robust call chain — rewind on failure); call backend `genpd->power_on / power_off`; measure latency and update `states[state_idx].power_{on,off}_latency_ns` if exceeded; dispatch `ON`/`OFF` notifier.

REQ-5: `genpd_runtime_suspend(dev)`: validate `dev_to_genpd(dev)`; consult `gov->suspend_ok(dev)` (PM-QoS check); call `__genpd_runtime_suspend(dev)` (walks subsystem hierarchy for actual `runtime_suspend` cb); call `genpd_stop_dev(genpd, dev)` (per-device backend stop hook). Then under `genpd_lock`: `genpd_power_off(genpd, true, 0)` + `genpd_drop_performance_state(dev)`. If IRQ-safe device in non-IRQ-safe domain: skip the genpd_power_off (the domain stays on).

REQ-6: `genpd_runtime_resume(dev)`: under `genpd_lock`: `genpd_power_on(genpd, 0)` + `genpd_restore_performance_state(dev, rpm_pstate)`. Then `genpd_start_dev(genpd, dev)` (per-device backend start) + `__genpd_runtime_resume(dev)`.

REQ-7: `pm_genpd_add_device(genpd, dev)`: allocate `generic_pm_domain_data`; install `dev->pm_domain = &genpd->domain`; link into `genpd->dev_list`; increment `genpd->device_count`. Refuse if device already attached or `genpd->prepared_count > 0`.

REQ-8: Performance-state aggregation: `_genpd_reeval_performance_state` walks `genpd->dev_list` and takes max of per-device `performance_state`; calls backend `set_performance_state(genpd, max)`; on parent link with non-zero performance: `_genpd_set_parent_state` recurses to propagate.

REQ-9: System PM: `genpd_prepare(dev)` increments `prepared_count` to block runtime-PM power-off; `_complete` decrements. `_suspend_noirq` / `_resume_noirq` use `genpd_sync_power_{off,on}` (unlocked since dpm_resume_noirq_list is serialized).

REQ-10: `pm_genpd_remove(genpd)`: refuse if `device_count > 0` or `sd_count > 0` or `genpd->prepared_count > 0`. Remove from `gpd_list`; free `gd`; release IDA-id.

REQ-11: Notifier chain: per-domain `genpd->power_notifiers` is `raw_notifier_head`; consumers register via `dev_pm_genpd_add_notifier(dev, nb)`; events delivered by `_genpd_power_on / _off` in PRE → ACT order with robust-rewind on PRE failure.

REQ-12: Debugfs: `<debugfs>/pm_genpd/<name>/{current_state,active_time,total_idle_time,sub_domains,devices,...}` exposed under `genpd_debugfs_dir` with read-only access (changes require CAP_SYS_ADMIN).

## Acceptance Criteria

- [ ] AC-1: `pm_genpd_init(genpd, &simple_qos_governor, true)` + `pm_genpd_add_device(genpd, dev)` + `pm_runtime_enable(dev)` + `pm_runtime_get_sync(dev)` results in `_power_on` called exactly once and `genpd->status == GENPD_STATE_ON`.
- [ ] AC-2: Last `pm_runtime_put(dev)` in the domain triggers `_power_off` exactly once; `genpd->status == GENPD_STATE_OFF`.
- [ ] AC-3: Two devices on same domain with conflicting QoS latency: `suspend_ok` consults the strictest constraint; `power_down_ok` only returns true if all are OK.
- [ ] AC-4: Subdomain test: child with `_RPM_ALWAYS_ON` keeps parent on even when child has no devices attached.
- [ ] AC-5: System suspend: `genpd_prepare` blocks runtime-PM power-off; `noirq` phase still transitions every non-always-on domain.
- [ ] AC-6: Notifier PRE_ON failure rewinds: subscribers that successfully observed PRE_ON receive PRE_ON-FAIL/OFF notification.
- [ ] AC-7: Concurrent attach/detach with `pm_genpd_remove` correctly refuses with `-EBUSY` if `device_count > 0`.

## Architecture

`Core` lives in `drivers::pmdomain::core`:

```
struct Domain {
  dev: KDevice,                       // genpd->dev (parented under genpd_provider_bus)
  domain: DevPmDomain,                // installed into dev->pm_domain on attach
  gpd_list_node: ListNode,
  parent_links: List<Link>,           // links where THIS domain is parent
  child_links: List<Link>,            // links where THIS domain is child
  dev_list: List<PmDomainData>,
  gov: Option<&'static Governor>,
  gd: Option<KBox<GovernorData>>,
  power_off_work: WorkStruct,
  provider: Option<&FwnodeHandle>,
  has_provider: bool,
  name: KStr,
  sd_count: AtomicU32,
  status: GpdStatus,                  // ON / OFF
  device_count: u32,
  device_id: u32,
  suspended_count: u32,
  prepared_count: u32,
  performance_state: u32,
  cpus: CpuMask,
  synced_poweroff: bool,
  stay_on: bool,
  sync_state: GenpdSyncState,
  power_off: Option<fn(&Domain) -> Result<()>>,
  power_on: Option<fn(&Domain) -> Result<()>>,
  power_notifiers: RawNotifierHead,
  opp_table: Option<&OppTable>,
  set_performance_state: Option<fn(&Domain, u32) -> Result<()>>,
  dev_ops: GpdDevOps,                 // start / stop
  attach_dev: Option<fn(&Domain, &Device) -> Result<()>>,
  detach_dev: Option<fn(&Domain, &Device)>,
  flags: u32,
  states: KBox<[PowerState]>,
  state_count: u32,
  state_idx: u32,
  on_time: u64,
  accounting_time: u64,
  lock_ops: &'static LockOps,
  lock: LockUnion,                    // mutex / spinlock / raw_spinlock per lock_ops
}
```

Power-on path `Core::power_on(genpd, depth)`:
1. `genpd_lock_nested(genpd, depth)`.
2. If `status == ON`: unlock + return 0 (idempotent).
3. For each `link` in `child_links`: lock parent (depth+1), recurse `power_on(parent, depth+1)`, increment `parent->sd_count`, unlock parent.
4. On any parent failure: roll back already-incremented parents.
5. `_genpd_power_on(genpd, timed=true)`: PRE_ON notifier → backend `power_on(genpd)` → measure latency → ON notifier.
6. `genpd->status = ON`.
7. Update accounting.

Power-off path `Core::power_off(genpd, one_dev_on, depth)`:
1. Refusal checks (status, prepared, flags, sd_count, child-deepest, device-suspended).
2. Governor `gov->power_down_ok(&genpd->domain)` consultation.
3. `_genpd_power_off(genpd, timed=true)`: PRE_OFF notifier → backend `power_off(genpd)` → measure latency → OFF notifier.
4. `genpd->status = OFF`; `states[state_idx].usage++`.
5. For each `link` in `child_links`: decrement `parent->sd_count`; recurse `power_off(parent, depth+1)`.

Runtime-PM hook `Core::runtime_suspend(dev)`:
1. `genpd = dev_to_genpd(dev)`.
2. `gov->suspend_ok(dev)` — QoS check; if false, return -EBUSY.
3. `__genpd_runtime_suspend(dev)` — walks subsystem hierarchy (type / class / bus / driver) for the actual `runtime_suspend` cb.
4. `genpd_stop_dev(genpd, dev)` — per-device backend stop.
5. Measure latency; update `td->suspend_latency_ns` if exceeded.
6. If non-IRQ-safe domain hosting IRQ-safe device: bail (keep domain on).
7. Under genpd_lock: `genpd_power_off(genpd, one_dev_on=true, 0)` + `rpm_pstate = drop_performance_state(dev)`.

System suspend `Core::suspend_noirq(dev)`:
1. `genpd_finish_suspend(dev, /poweroff=*/false)` — call subsystem `suspend_noirq` cb.
2. `genpd_sync_power_off(genpd, use_lock=false, depth=0)`.

Attach `Core::add_device(genpd, dev)`:
1. Refuse if `genpd->prepared_count > 0` (block during system suspend).
2. Refuse if `dev->pm_domain != NULL` (already attached).
3. `genpd_alloc_dev_data(dev, ...)`: alloc `generic_pm_domain_data` + `gpd_timing_data` td; initialize per-device QoS notifier.
4. `dev->pm_domain = &genpd->domain` (this is what re-routes runtime-PM).
5. `list_add_tail(&pdd->list_node, &genpd->dev_list)`.
6. `genpd->device_count++`.
7. `dev_pm_qos_add_notifier(dev, &gpd_data->nb)` so QoS changes notify the governor.

## Hardening

- Lock-ops chosen at init based on `GENPD_FLAG_IRQ_SAFE`; an IRQ-safe domain (raw spinlock) cannot host non-IRQ-safe devices in a way that would sleep under spinlock.
- `genpd_lock_nested` uses lockdep-tracked nesting depth, refuses excess depth.
- Robust notifier chain rewinds on PRE-failure so subscribers cannot see half-applied transitions.
- `prepared_count` blocks runtime-PM during system suspend.
- `_genpd_power_off`'s refuse-checks (`!status_on`, `prepared_count`, `ALWAYS_ON`, `sd_count`, child-deepest-state, device-rpm_always_on) are all cheap; the governor consultation comes last.
- Latency measurement is opt-in via `genpd->gd && !states[idx].fwnode` (firmware-provided latencies are authoritative).

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `generic_pm_domain`, `gpd_link`, `generic_pm_domain_data`, `gpd_timing_data`, `genpd_governor_data`, `genpd_power_state[]`; debugfs / sysfs prints length-bounded.
- **PAX_KERNEXEC** — `core.c` + `governor.c` text W^X; `genpd_mtx_ops` / `_spin_ops` / `_raw_spin_ops` lock-ops tables, governor vtables, and per-genpd `dev_pm_domain` ops in `__ro_after_init` cells.
- **PAX_RANDKSTACK** — randomize kernel stack offset across `genpd_power_on`, `_power_off`, `_runtime_suspend`, `_runtime_resume`, `_suspend_noirq`, `_set_performance_state`, governor entry points.
- **PAX_REFCOUNT** — saturating `refcount_t` on `genpd->device_count`, `genpd->sd_count`, `prepared_count`; saturating `kref` on `generic_pm_domain_data`; overflow trap defeats double-attach / detach-while-busy race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `generic_pm_domain_data`, `gpd_timing_data`, `genpd_governor_data`, per-state stats counters, and PRE-failure-rolled-back save-state buffers so stale ownership cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every debugfs / sysfs entry into genpd; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `dev_pm_domain` ops, backend `power_on`, `power_off`, `set_performance_state`, `attach_dev`, `detach_dev`, `set_hwmode_dev`, `get_hwmode_dev`, `dev_ops.start/stop`, governor `power_down_ok` / `system_power_down_ok` / `suspend_ok`, and notifier-block `notifier_call` indirect dispatch kCFI-typed with `__ro_after_init` storage.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `genpd_power_on / _off`, backend callbacks, governor entries behind CAP_SYSLOG; suppress `%p` in genpd tracepoints.
- **GRKERNSEC_DMESG** — restrict latency-exceeded banners, rejection counters, governor traces, and notifier-chain rewind logs to CAP_SYSLOG.
- **`pm_genpd_data` PAX_REFCOUNT** — saturating refcount on per-attached-device `generic_pm_domain_data`; double-detach and detach-while-prepared refused with -EBUSY.
- **RPM transition lock** — `genpd_lock` (mutex/spin/raw per IRQ-safe flag) wraps every runtime-PM state mutation; `genpd_lock_nested` enforces lockdep-tracked nesting bound.
- **Save/restore state PAX_USERCOPY** — `gpd_data->rpm_pstate` and per-state `usage`/`rejected` counters tracked in whitelisted slabs; sysfs/debugfs exposure length-bounded.
- **Governor swap CAP_SYS_ADMIN** — replacing `genpd->gov` (debugfs/sysfs path) requires CAP_SYS_ADMIN in the init user namespace; refuse swap while `genpd->prepared_count > 0`.
- **Performance-state aggregation bounded** — `_genpd_reeval_performance_state` walks bounded `dev_list`; refuse aggregation if `dev_list` length exceeds platform-supported cap.
- **Notifier rewind audited** — PRE_ON/PRE_OFF failure rewind logged at trace-level with CAP_SYSLOG gating; refuse to skip rewind on partial failure.

Rationale: the genpd core is the sole arbiter of when SoC power rails turn on/off and at what performance state, including rails powering TPM, secure-EL3 mailbox, JTAG/secure-debug, and crypto IP. A refcount underflow on `device_count`, a missed CAP_SYS_ADMIN check on governor swap, a corrupted backend `power_off` callback, or an unbounded `dev_list` walk can keep secure-only rails on, defeat S3 wake, or panic via lockdep depth. Refcount saturation, kCFI on every callback table, CAP_SYS_ADMIN on governor mutations, RPM-transition-lock auditing, and notifier-rewind verification turn the genpd core into a structural power-policy enforcement boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `power_on_off_idempotent` | INVARIANT | `genpd_power_on` on already-on domain is a no-op; same for `_power_off` on already-off. |
| `device_count_no_underflow` | OVERFLOW | `genpd->device_count` saturates; refuses remove below zero. |
| `sd_count_no_underflow` | OVERFLOW | `genpd->sd_count` saturates; refuses negative. |
| `state_idx_no_oob` | OOB | every `states[state_idx]` access has `state_idx < state_count`. |
| `dev_list_walk_no_uaf` | UAF | `dev_list` walks under `genpd_lock`; no concurrent free of `pm_domain_data`. |

### Layer 2: TLA+

`models/pmdomain/genpd_runtime.tla`: proves that under arbitrary interleavings of `pm_runtime_get_sync` and `pm_runtime_put`, the backend `power_on` is called iff a transition `OFF → ON` happens, and `power_off` iff `ON → OFF`. No spurious calls.

`models/pmdomain/notifier_rewind.tla`: proves PRE_ON failure rewinds correctly — every subscriber that received PRE_ON also receives the PRE_ON-FAIL/OFF notification, in the reverse order.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `genpd_power_on` post: `status == ON` AND `sd_count++` for every parent | `Core::power_on` |
| `genpd_power_off` post: `status == OFF` AND `sd_count--` for every parent AND `states[idx].usage++` | `Core::power_off` |
| `pm_genpd_add_device` post: `dev->pm_domain == &genpd->domain` AND device count incremented | `Core::add_device` |
| `_genpd_set_performance_state` post: `genpd->performance_state == max(per-device requests)` | `Core::set_perf` |

### Layer 4: Verus/Creusot functional

Full lifecycle: `pm_genpd_init → add_device(x N) → pm_runtime_get_sync(each) → set_performance_state(each) → pm_runtime_put(each) → remove_device(x N) → pm_genpd_remove` leaves zero refcounts, no leaked `pm_domain_data`, no dangling `dev->pm_domain`, no stuck `genpd->status`.

## Out of Scope

- Per-vendor genpd backends (covered in future per-vendor `drivers/pmdomain/{vendor}/*.md`)
- DT provider helpers `of_genpd_*` (future `genpd-of.md`)
- `cpuidle` integration internals (future `cpuidle-core.md`)
- OPP table / DVFS integration (future `opp-core.md`)
- 32-bit-only paths
- Implementation code
