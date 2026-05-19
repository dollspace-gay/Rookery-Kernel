# Tier-3: drivers/phy/phy-core.c — Generic PHY framework core (provider register, lookup, init/power refcounts)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/phy/00-overview.md
upstream-paths:
  - drivers/phy/phy-core.c
  - include/linux/phy/phy.h
-->

## Summary

`phy-core.c` is the framework heart of `drivers/phy/`: ~1300 lines that provide the consumer/provider matching machinery (`of_phy_get`, `phy_get`, `phy_lookup`), the per-PHY lifecycle helpers (`phy_init`, `phy_power_on`, `phy_set_mode_ext`, `phy_configure`, `phy_validate`, `phy_calibrate`, `phy_reset`, `phy_notify_connect/disconnect/state`), the allocator (`phy_create` / `devm_phy_create` / `phy_destroy`), and the OF / non-OF provider registration (`__of_phy_provider_register`, `devm_*` variants, `phy_create_lookup`). All vendor PHY drivers in `drivers/phy/<vendor>/` are clients of this core.

This Tier-3 covers `phy-core.c` exclusively + the consumer-facing parts of `include/linux/phy/phy.h`. Vendor subdir drivers (Qualcomm QMP, Rockchip Type-C, etc.) inherit `phy_ops` semantics from the contract in this doc but are out of scope.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `static const struct class phy_class` | `/sys/class/phy/` class with `dev_release = phy_release` | `Subsystem::phy_class` |
| `static struct dentry *phy_debugfs_root` | `debugfs/phy/` root | `Subsystem::debugfs_root` |
| `static DEFINE_MUTEX(phy_provider_mutex)` | guards `phy_provider_list` + `phys` lookup-list + `of_xlate` | `Subsystem::provider_mutex` |
| `static LIST_HEAD(phy_provider_list)` | all registered `phy_provider`s | `Subsystem::provider_list` |
| `static LIST_HEAD(phys)` | non-OF `phy_lookup` entries | `Subsystem::lookup_list` |
| `static DEFINE_IDA(phy_ida)` | per-PHY id space | `Subsystem::ida` |
| `phy_create(dev, node, &ops)` | allocate `struct phy` + class device + ida + debugfs subdir | `Phy::create` |
| `devm_phy_create(dev, node, &ops)` | devres-managed phy_create | `Phy::create_devm` |
| `phy_destroy(phy)` | unregister class device + drop ida + free | `Phy::destroy` |
| `phy_create_lookup(phy, con_id, dev_id)` / `phy_remove_lookup(...)` | non-OF lookup table entry | `Phy::create_lookup` / `Phy::remove_lookup` |
| `phy_find(dev, con_id)` | match `phy_lookup` by dev_name + con_id | `Phy::find_by_dev` |
| `_of_phy_get(np, index)` / `of_phy_get(np, con_id)` | OF-based lookup (parses `phys = <...>` + invokes provider `of_xlate`) | `Phy::of_get` |
| `phy_get(dev, con_id)` / `devm_phy_get(dev, con_id)` / `phy_put(dev, phy)` | top-level consumer entry points | `Phy::get` / `Phy::get_devm` / `Phy::put` |
| `__of_phy_provider_register(dev, children, owner, of_xlate)` / `of_phy_provider_unregister(provider)` | provider lifecycle | `PhyProvider::register` / `_unregister` |
| `__devm_of_phy_provider_register(...)` / `devm_of_phy_provider_unregister(dev, provider)` | devres-managed provider lifecycle | `PhyProvider::register_devm` |
| `phy_init(phy)` / `phy_exit(phy)` | per-PHY init refcount path | `Phy::init` / `Phy::exit` |
| `phy_power_on(phy)` / `phy_power_off(phy)` | per-PHY power refcount path (regulator-aware) | `Phy::power_on` / `Phy::power_off` |
| `phy_pm_runtime_get / _get_sync / _put / _put_sync` | wrapper around per-PHY runtime PM | `Phy::pm_runtime_*` |
| `phy_set_mode_ext(phy, mode, submode)` / `phy_set_media` / `phy_set_speed` | runtime reconfig | `Phy::set_*` |
| `phy_configure(phy, opts)` / `phy_validate(phy, mode, submode, opts)` | typed configuration / dry-run | `Phy::configure` / `Phy::validate` |
| `phy_calibrate(phy)` / `phy_reset(phy)` | trim + reset | `Phy::calibrate` / `Phy::reset` |
| `phy_notify_connect / _disconnect / _state` | consumer-driven event hooks | `Phy::notify_*` |

## Compatibility contract

REQ-1: `phy_create` MUST allocate a `struct phy` with `device_initialize`, `ida_alloc(&phy_ida)`, `dev_set_name(dev, "phy-%s.%d", ...)`, `lockdep_register_key(&phy->lockdep_key)`, `mutex_init_with_key(&phy->mutex)`, and `device_add` under `phy_class`. Failure releases ida + lockdep key + frees.

REQ-2: `phy_destroy` MUST `pm_runtime_disable(&phy->dev)` then `device_unregister(&phy->dev)`; the `phy_release` callback (kobject release) then drops regulator, mutex, lockdep key, and ida.

REQ-3: `of_phy_get` parses `np->phys` phandle list via `of_parse_phandle_with_args(np, "phys", "#phy-cells", index, &args)`. If the target node is `compatible = "usb-nop-xceiv"`, return `-ENODEV` (legacy USB PHY subsystem owns it).

REQ-4: Provider lookup `of_phy_provider_lookup` matches `provider->dev->of_node == args.np` first, then walks `provider->children` for nested PHY nodes via `for_each_child_of_node_scoped`. No match → `-EPROBE_DEFER`.

REQ-5: `provider->of_xlate(provider->dev, &args)` MUST be called under `phy_provider_mutex` AND with `try_module_get(provider->owner)` succeeded; failure of `try_module_get` returns `-EPROBE_DEFER`.

REQ-6: `phy_init` / `phy_exit` / `phy_power_on` / `phy_power_off` are NOT callable from atomic context: each takes `phy->mutex` (sleepable) AND may call `pm_runtime_get_sync` (sleepable).

REQ-7: `phy_init` MUST call `pm_runtime_get_sync` BEFORE acquiring `phy->mutex`; on `ops->init` failure, drop `pm_runtime_put` AFTER releasing mutex. The pair preserves "PM ref held during driver op."

REQ-8: `phy_power_on` MUST enable `phy->pwr` (regulator) BEFORE `pm_runtime_get_sync`; rollback path disables regulator on `pm_runtime_get_sync` failure AND on `ops->power_on` failure.

REQ-9: `init_count` and `power_count` are integer refcounts protected by `phy->mutex`. `ops->init` invoked only on 0→1 transition of `init_count`; `ops->exit` invoked only on 1→0. Same for power.

REQ-10: `dev_warn` fired if `phy_power_on` is called when `power_count > init_count` (consumer called power before init).

REQ-11: `phy_set_mode_ext` updates `phy->attrs.mode` ONLY if `ops->set_mode` returned 0; on success the new mode is observable via debugfs / sysfs.

REQ-12: `phy_configure` returns `-EOPNOTSUPP` if `ops->configure == NULL`; `phy_validate` returns `-EOPNOTSUPP` if `ops->validate == NULL`. `phy_validate` MUST NOT mutate state.

REQ-13: `phy_create_lookup` / `phy_remove_lookup` operate on the global `phys` list under `phy_provider_mutex`; entries are `{phy *, con_id, dev_id}` tuples for non-OF / board-file consumers.

REQ-14: `devm_phy_get` registers a devres entry releasing the PHY via `phy_put` when the consumer device unbinds; `devm_phy_create` mirrors with `phy_destroy` on unbind.

## Acceptance Criteria

- [ ] AC-1: `class_register(&phy_class)` succeeds at `device_initcall` boot phase; `/sys/class/phy/` exists.
- [ ] AC-2: `debugfs_create_dir("phy", NULL)` creates `/sys/kernel/debug/phy/`; non-fatal on debugfs disable.
- [ ] AC-3: Two-consumer test on shared QMP USB+DP PHY: both consumers' `phy_init` returns 0; `ops->init` invoked exactly once; `init_count == 2` observable via debugfs after second call.
- [ ] AC-4: Provider unregister during outstanding consumer ref refused by `try_module_get` semantics (consumer holds module ref via `of_phy_get`).
- [ ] AC-5: `of_phy_get` returns `-EPROBE_DEFER` when provider not yet registered; consumer probe reruns after provider binds.
- [ ] AC-6: `phy_power_on` regulator-enable-fail path: `ops->power_on` not invoked; `power_count` unchanged; regulator not left enabled.
- [ ] AC-7: `phy_set_mode_ext` failure leaves `phy->attrs.mode` at previous value (regression test).
- [ ] AC-8: `phy_validate` on a phy without `ops->validate` returns `-EOPNOTSUPP` without mutex contention.
- [ ] AC-9: `devm_phy_get` + consumer-device unbind frees the PHY ref automatically (verified via /sys disappearance).

## Architecture

`phy-core.c` is structured in four functional bands:

1. **Consumer lookup** (~lines 60-180, 610-900): `phy_create_lookup`, `phy_remove_lookup`, `phy_find`, `_of_phy_get`, `of_phy_get`, `phy_get`, `devm_phy_get`, `phy_put`, `devm_phy_put`.
2. **Lifecycle ops** (~lines 150-700): `phy_pm_runtime_*`, `phy_init`, `phy_exit`, `phy_power_on`, `phy_power_off`, `phy_set_mode_ext`, `phy_set_media`, `phy_set_speed`, `phy_reset`, `phy_calibrate`, `phy_notify_connect`, `phy_notify_disconnect`, `phy_notify_state`, `phy_configure`, `phy_validate`.
3. **Object allocator** (~lines 900-1130): `phy_create`, `devm_phy_create`, `phy_destroy`, `devm_phy_destroy`, `phy_release`.
4. **Provider lifecycle** (~lines 1130-1310): `__of_phy_provider_register`, `__devm_of_phy_provider_register`, `of_phy_provider_unregister`, `devm_of_phy_provider_unregister`, `phy_core_init`, `phy_core_exit`.

The Rookery analog `drivers::phy::core` has the same four modules; `Subsystem` carries `phy_class`, `provider_mutex`, `provider_list`, `lookup_list`, `ida`, `debugfs_root`. All globals are initialized in `phy_core_init` (device_initcall priority).

### init/power refcount machinery

```
fn phy_init(phy: &Phy) -> Result<()> {
    pm_runtime_get_sync(phy)?;
    let guard = phy.mutex.lock();
    if phy.power_count.load() > phy.init_count.load() {
        warn!("phy_power_on was called before phy_init");
    }
    if phy.init_count.load() == 0 {
        if let Some(init) = phy.ops.init {
            init(phy)?;     // any error rolls back without touching count
        }
    }
    phy.init_count.fetch_add(1);
    drop(guard);
    pm_runtime_put(phy);
    Ok(())
}
```

`phy_power_on` is structurally the same plus a `regulator_enable(phy.pwr)` prefix and a `regulator_disable` suffix on the error/exit edges. Crucially, the runtime-PM ref is released even on success — the per-call ref is only needed to ensure the driver `init/power_on` op runs with PM active; long-term PM is the driver's own concern.

### Provider lookup under mutex

```
fn _of_phy_get(np: &DeviceNode, index: usize) -> Result<*Phy> {
    let args = of_parse_phandle_with_args(np, "phys", "#phy-cells", index)?;
    if of_device_is_compatible(args.np, "usb-nop-xceiv") {
        return Err(ENODEV);
    }
    let _guard = provider_mutex.lock();
    let provider = of_phy_provider_lookup(args.np)?;     // EPROBE_DEFER if absent
    let _module = try_module_get(provider.owner).ok_or(EPROBE_DEFER)?;
    if !of_device_is_available(args.np) {
        return Err(ENODEV);
    }
    provider.of_xlate(provider.dev, &args)
}
```

`provider->of_xlate` is the vendor-specific cell-decoder. Typical Qualcomm QMP `of_xlate` is `of_phy_simple_xlate` which uses `args.args[0]` as the lane index and returns that lane's `struct phy *`.

### Object allocation

`phy_create` builds the `struct phy` + `struct device` then calls `device_add(&phy->dev)`. Because `dev_set_name` uses a `phy-<dev_name>.<id>` pattern, multiple PHYs under one provider get unique names. The `lockdep_register_key(&phy->lockdep_key)` + `mutex_init_with_key(&phy->mutex, ...)` pairing gives each PHY its own lockdep class to avoid shared-mutex false positives across the framework.

### Provider registration

`__of_phy_provider_register` validates that the `children` arg (when non-NULL) is the provider's `of_node` or a descendant — climbs the parent chain looking for a match, refuses with `-EINVAL` otherwise. This prevents a buggy driver from registering as the provider for an unrelated DT subtree.

`devm_*` variants wrap with `devres_alloc(devm_phy_provider_release, ...)` so unbind reliably calls `of_phy_provider_unregister`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lookup_list_no_uaf` | UAF | `phys` list traversal in `phy_find` always under `phy_provider_mutex`; insertion + removal serialized |
| `provider_list_no_uaf` | UAF | `phy_provider_list` traversal in `of_phy_provider_lookup` always under `phy_provider_mutex` |
| `init_count_no_overflow` | OVERFLOW | `init_count` bounded by Rookery's `refcount_t::MAX`; saturating add |
| `power_count_no_overflow` | OVERFLOW | `power_count` bounded by Rookery's `refcount_t::MAX`; saturating add |
| `regulator_balanced` | UAF | every `regulator_enable` in `phy_power_on` paired with `regulator_disable` on every error edge |
| `module_get_balanced` | UAF | `try_module_get(provider.owner)` paired with `module_put(provider.owner)` on every `_of_phy_get` exit |
| `phy_release_no_use_after_free` | UAF | `phy_release` runs only after last `put_device(&phy->dev)`; consumer holds `get_device` until `phy_put` |
| `args_count_bounded` | OOB | `of_parse_phandle_with_args` returns `args.args_count <= MAX_PHANDLE_ARGS`; vendor `of_xlate` validates `args.args_count` against `#phy-cells` |

### Layer 2: TLA+

`models/phy/refcount.tla` (parent-declared): models N consumers each calling `init/exit/power_on/power_off` in arbitrary interleavings. Properties: (a) `ops->init` invoked exactly once per init_count 0→1 transition, (b) `ops->exit` invoked exactly once per init_count 1→0, (c) at quiescence `init_count == 0 ∧ power_count == 0 → no driver op outstanding`.

`models/phy/lookup_race.tla`: models provider register + consumer lookup + provider unregister interleaving. Property: consumer never gets a `phy *` whose owning module has unloaded.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `phy_init` post: `init_count_new == init_count_old + 1 ∧ (init_count_old == 0 → ops.init invoked)` | `Phy::init` |
| `phy_exit` post: `init_count_new == init_count_old - 1 ∧ (init_count_new == 0 → ops.exit invoked)` | `Phy::exit` |
| `phy_power_on` post: `pwr` regulator is enabled iff `power_count > 0 ∨ (return == Err ∧ no pwr ever enabled)` | `Phy::power_on` |
| `phy_set_mode_ext` post: `attrs.mode == mode ↔ ops.set_mode returned 0` | `Phy::set_mode` |
| `_of_phy_get` post: returned `phy *` belongs to `provider_list` ∧ caller holds `module_get(provider.owner)` | `Phy::of_get` |

### Layer 4: Verus/Creusot functional

Across DT-driven boot: provider DT node + consumer DT node + `phys = <&prov 0>;` => after `phy_core_init` + provider probe + consumer probe, consumer's `phy_init` returns Ok and `ops->init` is observably invoked exactly once. Encoded as a Verus refinement: starting from `Subsystem::empty()`, applying `provider.register()` then `consumer.get_and_init()` yields a state where `ops.init.call_count == 1`.

## Hardening

(Inherits from `drivers/phy/00-overview.md` § Hardening.)

`phy-core.c`-specific reinforcement:

- **`phy_provider_mutex` held across `of_xlate`** — defense against provider unregister mid-lookup.
- **`try_module_get(provider->owner)` before `of_xlate` invocation** — defense against module unload race.
- **`phy_release` deferred to last `put_device`** — defense against UAF when consumer still holds a reference at provider tear-down (consumer's `get_device` keeps it alive).
- **`children` validation in `__of_phy_provider_register`** — defense against a driver claiming PHY-provider duties for an unrelated DT subtree.
- **`dev_warn` when `power_count > init_count`** — defense against driver-order bug; surfaces misuse early.
- **Power-on rollback on every error edge** — `err_pwr_on` / `err_pm_sync` / `out` labels ensure regulator + PM ref balanced.
- **`phy_set_mode_ext` only updates `attrs.mode` on success** — defense against stale-mode misreporting.
- **`phy_validate` documented to be side-effect-free** — drivers MUST NOT mutate hardware; framework cannot enforce but Rookery wrapper Kani-asserts.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `struct phy`, `phy_provider`, `phy_lookup`, and the per-mode `phy_configure_opts` payloads (MIPI D-PHY, DP, LVDS, HDMI); refuse direct user-buffer copies into PHY core.
- **PAX_KERNEXEC** — `phy-core.c` text + global ops dispatch table live in W^X kernel text; `phy_ops`, `phy_provider->of_xlate`, the `phy_class` dev_release, and the `uio_logical_vm_ops`-style helpers all `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `phy_init`, `phy_power_on`, `phy_set_mode_ext`, `phy_configure`, `_of_phy_get`, `__of_phy_provider_register`, and `phy_release`.
- **PAX_REFCOUNT** — migrate `phy.init_count` and `phy.power_count` from raw `int` to saturating `refcount_t`; the existing `phy->mutex` becomes the atomicity guard but overflow / underflow now traps instead of wrapping.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `struct phy` (in `phy_release`), `phy_provider` (in `of_phy_provider_unregister`), and `phy_lookup` (in `phy_remove_lookup`) so stale mode / count / regulator pointers cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every consumer entry path (USB ioctl, DRM modeset, PCIe sysfs) that lands in `phy-core.c`; reject user-pointer deref outside canonical `copy_*_user` helpers.
- **PAX_RAP / kCFI** — `phy_ops` (`init`, `exit`, `power_on`, `power_off`, `set_mode`, `set_media`, `set_speed`, `configure`, `validate`, `reset`, `calibrate`, `connect`, `disconnect`, `notify_phystate`, `release`) and `phy_provider->of_xlate` marked `__ro_after_init` with kCFI-typed indirect dispatch; the `phy_class.dev_release = phy_release` slot also kCFI-typed.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-PHY private pointer disclosure behind CAP_SYSLOG; suppress `%p` in `dev_dbg`/`dev_vdbg` for `phy->id`, `phy->ops`, and `phy_provider->dev->of_node` traces.
- **GRKERNSEC_DMESG** — restrict PHY init-fail, power-on-fail, regulator-fail, and "phy_power_on was called before phy_init" banners to CAP_SYSLOG.
- **mutex-protected init counter migration** — the audit-recommended move to `refcount_t` keeps `phy->mutex` as the call-serialization mechanism but eliminates the racy `++init_count` / `--init_count` integer arithmetic; underflow traps catch double-exit bugs in vendor drivers.
- **fwnode lookup gated** — `of_phy_provider_lookup` validated against the active platform's DT/fwnode root; defense against a malicious DT overlay claiming the provider role for an in-use node.
- **CAP_SYS_ADMIN strict** — debugfs control under `/sys/kernel/debug/phy/` requires CAP_SYS_ADMIN; live mode-switching via `/sys/class/phy/phy-N/mode` (where exposed) gated.
- **of_xlate args bounded** — wrapper rejects `args.args_count > provider.cells_count`; vendor `of_xlate` receives a typed view with per-cell range checks (no raw `args.args[0]` access).
- **regulator-disable on every error edge audited** — Verus invariant: `phy_power_on::return == Err → !regulator_is_enabled(phy.pwr)`.
- **lockdep key per PHY** — `lockdep_register_key(&phy->lockdep_key)` ensures false-positive cycles across shared mutex use by N consumers don't mask real bugs.

Rationale: `phy-core.c` is small but it's the gatekeeper for every PCIe, USB, SATA, DSI, DP, and HDMI link on the system. A refcount underflow can leave the PHY off while a consumer believes it's on (data corruption on the wire). A racy provider lookup can hand a consumer a `phy *` whose module is unloading (UAF). RAP/kCFI on `phy_ops` + saturating refcounts + fwnode-gated lookup + scoped regulator/PM balance turn this thin coordination layer into a verified contract rather than a polite suggestion.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Vendor PHY drivers (`drivers/phy/<vendor>/`): future per-driver Tier-3 docs.
- `phy-core-mipi-dphy.c`: future `phy-mipi-dphy-core.md` Tier-3.
- `phy-common-props.c` / `phy-common-props-test.c`: future `phy-common-props.md` Tier-3.
- Legacy USB PHY framework (`drivers/usb/phy/`): tracked under `usb/`.
- Network PHY MDIO bus (`drivers/net/phy/`): separate subsystem.
- Implementation code.
