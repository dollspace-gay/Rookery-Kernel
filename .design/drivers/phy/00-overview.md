# Tier-3: drivers/phy/ — Generic PHY subsystem (provider/consumer, init/power-on/set-mode lifecycle)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/phy/
  - include/linux/phy/phy.h
-->

## Summary

The generic PHY (physical layer) framework is the kernel's abstraction for analog/mixed-signal PHY blocks that sit between a controller (USB host, PCIe RC, SATA HBA, MIPI DSI, Ethernet MAC, UFS, DisplayPort) and the off-chip lane / pad / serdes. A PHY is a separate device-tree node — `phy-provider` — registered by a vendor driver (Qualcomm QMP, TI Tegra, Rockchip Type-C, Samsung USB DRD, MediaTek T-PHY, Allwinner sun4i, Cadence Sierra, Synopsys eUSB2). Consumers (USB host stack, PCIe controller, DSI bridge) look up the PHY via `phy_get` / `of_phy_get` + drive its lifecycle (`phy_init` → `phy_power_on` → traffic → `phy_power_off` → `phy_exit`).

This overview Tier-3 spans the entire `drivers/phy/` tree: ~36 vendor subdirs (`allwinner/`, `amlogic/`, `apple/`, `broadcom/`, `cadence/`, `freescale/`, `hisilicon/`, `intel/`, `marvell/`, `mediatek/`, `qualcomm/`, `renesas/`, `rockchip/`, `samsung/`, `socionext/`, `st/`, `starfive/`, `tegra/`, `ti/`, `xilinx/`, ...) plus the framework core (`phy-core.c`, `phy-core-mipi-dphy.c`, `phy-common-props.c`). The core itself is covered by `drivers/phy/phy-core.md` Tier-3.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct phy` | per-PHY-device control block (init/power refcounts, ops, regulator) | `drivers::phy::Phy` |
| `struct phy_ops` | per-driver vtable: `init` / `exit` / `power_on` / `power_off` / `set_mode` / `set_media` / `set_speed` / `configure` / `validate` / `reset` / `calibrate` / `connect` / `disconnect` / `notify_phystate` / `release` | `drivers::phy::PhyOps` |
| `struct phy_provider` | per-DT-node provider record + `of_xlate` callback | `drivers::phy::PhyProvider` |
| `enum phy_mode` | mode discriminant (USB_HOST_SS, USB_DEVICE_HS, PCIE, SATA, ETHERNET, MIPI_DPHY, DP, HDMI, UFS_HS_A/B, LVDS) | `drivers::phy::PhyMode` |
| `union phy_configure_opts` | per-protocol mode-config payload (MIPI D-PHY, DP, LVDS, HDMI) | `drivers::phy::ConfigureOpts` |
| `phy_create(dev, node, &ops)` / `phy_destroy(phy)` | per-PHY alloc + class register | `Phy::create` / `Phy::destroy` |
| `__of_phy_provider_register(dev, children, owner, of_xlate)` / `of_phy_provider_unregister(provider)` | per-DT-node provider lifecycle | `PhyProvider::register` / `PhyProvider::unregister` |
| `of_phy_get(np, con_id)` / `phy_get(dev, con_id)` / `phy_put(dev, phy)` | consumer lookup + release | `Phy::get_of` / `Phy::get` / `Phy::put` |
| `phy_init(phy)` / `phy_exit(phy)` | per-PHY init / exit refcount-paired | `Phy::init` / `Phy::exit` |
| `phy_power_on(phy)` / `phy_power_off(phy)` | per-PHY power refcount-paired (with regulator) | `Phy::power_on` / `Phy::power_off` |
| `phy_set_mode_ext(phy, mode, submode)` / `phy_set_media(phy, media)` / `phy_set_speed(phy, speed)` | per-PHY runtime reconfig | `Phy::set_mode` / `Phy::set_media` / `Phy::set_speed` |
| `phy_configure(phy, opts)` / `phy_validate(phy, mode, submode, opts)` | per-protocol explicit config + dry-run validation | `Phy::configure` / `Phy::validate` |
| `phy_calibrate(phy)` / `phy_reset(phy)` | per-PHY trim + reset | `Phy::calibrate` / `Phy::reset` |
| `phy_notify_connect(phy, port)` / `phy_notify_disconnect(phy, port)` / `phy_notify_state(phy, state)` | per-PHY consumer-side notifications (USB connect, UFS Hibern8) | `Phy::notify_*` |

## Compatibility contract

REQ-1: Each PHY is a distinct `struct device` under the `phy` class, registered with `phy_class` + an IDA-allocated minor (`phy_ida`); the kobject path is `/sys/class/phy/phy-<id>`.

REQ-2: PHY providers register via OF (`of_xlate` callback maps `of_phandle_args` from the consumer's `phys = <&prov a b c>` DT entry to a specific PHY instance). Non-OF lookup via the `phy_lookup` table (`phy_create_lookup`) is supported for board-file / ACPI consumers.

REQ-3: Consumer lifecycle MUST be `phy_init` → `phy_power_on` → traffic → `phy_power_off` → `phy_exit`; `init` may sleep (PLL lock, regulator ramp); `power_on/off` may sleep. `phy_power_off` must be called before `phy_exit`.

REQ-4: `init_count` and `power_count` are reference-counted integers protected by `phy->mutex`; multiple consumers sharing one PHY pair their init/power calls. Only the first init / first power_on invokes the driver op; only the last exit / last power_off invokes the inverse.

REQ-5: `phy_set_mode_ext` may be called at any time after `phy_init` and before `phy_exit`; mode change MUST be atomic with respect to consumer's traffic stop.

REQ-6: `phy_configure` requires `phy_init` to have been called; opts payload is per-mode (`union phy_configure_opts`) — MIPI D-PHY uses `phy_configure_opts_mipi_dphy`, DP uses `phy_configure_opts_dp`, etc.

REQ-7: `phy_validate` must NOT mutate hardware state; consumers may call it iteratively to probe acceptable parameter ranges.

REQ-8: Module-owner refcounts: `try_module_get(phy->ops->owner)` is taken on `of_phy_get`; `phy_put` drops it. Providers also take `try_module_get(phy_provider->owner)` while the consumer iterates lookup.

REQ-9: Power regulator: optional `phy->pwr` is enabled before `power_on` op + disabled after `power_off`; failure to enable regulator aborts `power_on` and propagates `-EREGULATOR_*` to caller.

REQ-10: Runtime PM: `phy_pm_runtime_get_sync` is invoked at the boundary of init/power; PHYs that do not enable runtime PM return `-ENOTSUPP` which the framework swallows.

REQ-11: Atomic suspend / resume safe: PHY framework operations are not callable from atomic context (mutex + may-sleep regulator); `notify_connect` / `notify_disconnect` are sleepable too.

REQ-12: Provider lifetime: `of_node_get(children)` is held by the provider; `of_phy_provider_unregister` drops it. Consumers MUST drop their `module_put` + `put_device` via `phy_put`.

## Acceptance Criteria

- [ ] AC-1: USB3 SuperSpeed link on Qualcomm QMP USB3+DP combo PHY: `dwc3` consumer + `phy_qcom_qmp_combo` provider, lifecycle clean across hotplug.
- [ ] AC-2: PCIe Gen3 link on Rockchip RK3399 Type-C PHY: `pcie-rockchip` consumer + `phy_rockchip_typec`.
- [ ] AC-3: SATA on TI Keystone PHY: `ahci_dwc` consumer + Keystone SerDes provider.
- [ ] AC-4: MIPI D-PHY on Samsung Exynos DSI: DSIM consumer + `phy_exynos5_usbdrd` lane config via `phy_configure(opts.mipi_dphy)`.
- [ ] AC-5: DisplayPort link training on AMD/Intel APU: `amdgpu/i915` consumer + `phy_intel_lgm` / `phy_amd_pmc` providers via `phy_configure(opts.dp)`.
- [ ] AC-6: shared-PHY refcount: USB3 host + USB3 device sharing one combo PHY each call init/power_on; only first invokes driver `init`.
- [ ] AC-7: `/sys/class/phy/phy-N/mode` reflects current `phy->attrs.mode` after `phy_set_mode_ext`.
- [ ] AC-8: `phy_validate` returns 0/-EINVAL without altering PHY register state on supported drivers.
- [ ] AC-9: Module unload-while-in-use refused via `try_module_get` failure path.

## Architecture

The framework is a thin coordination layer; vendor drivers carry all hardware specifics. Three globals in `phy-core.c`:

```
static const struct class phy_class      // /sys/class/phy/
static DEFINE_IDA(phy_ida);              // per-PHY id allocator
static LIST_HEAD(phy_provider_list);     // all registered providers
static LIST_HEAD(phys);                  // non-OF phy_lookup entries
static DEFINE_MUTEX(phy_provider_mutex); // protects both lists
```

Consumer lookup path (`of_phy_get` → `_of_phy_get`):
1. `of_parse_phandle_with_args(np, "phys", "#phy-cells", index, &args)` extracts provider node + cell args.
2. `of_phy_provider_lookup(args.np)` walks `phy_provider_list` matching provider's `of_node` or any of its child nodes.
3. If provider not yet registered, returns `-EPROBE_DEFER` so consumer probes deferred.
4. `provider->of_xlate(provider->dev, &args)` invoked under `phy_provider_mutex` returns a `struct phy *` (vendor decides which lane / instance).
5. Consumer takes `try_module_get(phy->ops->owner)` + `get_device(&phy->dev)`.

Lifecycle (per-PHY, protected by `phy->mutex`):
- `phy_init`: `pm_runtime_get_sync` → if `init_count == 0` call `ops->init` → `++init_count` → `pm_runtime_put`.
- `phy_exit`: if `init_count == 1` call `ops->exit` → `--init_count`.
- `phy_power_on`: `regulator_enable(pwr)` → `pm_runtime_get_sync` → if `power_count == 0` call `ops->power_on` → `++power_count`.
- `phy_power_off`: if `power_count == 1` call `ops->power_off` → `--power_count` → `regulator_disable(pwr)`.
- `phy_set_mode_ext`: under `phy->mutex` invoke `ops->set_mode(phy, mode, submode)`; on success stash `phy->attrs.mode = mode`.

PHY object structure (`struct phy`):
```
struct phy {
    struct device dev;             // phy_class device, parent = provider dev
    int id;                        // ida_alloc'd
    const struct phy_ops *ops;     // driver vtable
    struct mutex mutex;            // protects {init,power}_count + ops calls
    struct lock_class_key lockdep_key;
    int init_count;                // refcount
    int power_count;               // refcount
    struct phy_attrs attrs;        // bus_width, max_link_rate, current mode
    struct regulator *pwr;         // optional VPHY rail
    struct dentry *debugfs;
};
```

Provider object (`struct phy_provider`):
```
struct phy_provider {
    struct device *dev;            // provider hardware device
    struct device_node *children;  // node containing child PHYs (default dev->of_node)
    struct module *owner;          // module containing of_xlate
    struct list_head list;
    struct phy *(*of_xlate)(struct device *dev, const struct of_phandle_args *args);
};
```

Subsystem init (`phy_core_init`, `device_initcall`):
1. `class_register(&phy_class)` — creates `/sys/class/phy/`.
2. `debugfs_create_dir("phy", NULL)` — creates `/sys/kernel/debug/phy/` for per-PHY debugfs subdirs.

Vendor driver shape (USB QMP example): probe parses DT lanes, calls `devm_phy_create` per lane with vendor `phy_ops`, then `devm_of_phy_provider_register` with an `of_xlate` that maps lane index to the right `struct phy *`. The DWC3 USB controller probe calls `devm_phy_get(dev, "usb3-phy")` → defers if QMP not yet bound → eventually obtains `phy *` → `phy_init/power_on/set_mode(USB_HOST_SS)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `phy_init_power_paired` | UAF | every `phy_init` consumer pair matched by `phy_exit`; init_count never underflows |
| `lookup_no_use_after_free` | UAF | `of_phy_get` returns either ERR_PTR or a phy with module ref + dev get |
| `of_xlate_args_bounded` | OOB | `of_xlate` receives `args.args_count` ≤ MAX_PHANDLE_ARGS (16); vendor xlate must validate per-cell range |
| `init_count_no_overflow` | OVERFLOW | shared PHYs: init_count + power_count overflow rejected via REFCOUNT semantics |

### Layer 2: TLA+

`models/phy/lifecycle.tla` (parent-declared): proves that for any interleaving of N consumers calling `init/power_on/power_off/exit` on a shared PHY, driver `ops->init` is called exactly once for each (0→1) transition and `ops->exit` exactly once on (1→0).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Phy::init` post: `phy.init_count > 0 ∧ ops.init invoked iff (prev init_count == 0)` | `Phy::init` |
| `Phy::power_on` post: regulator enabled implies `power_count > 0`; on error path regulator disabled | `Phy::power_on` |
| `set_mode` post: `phy.attrs.mode == mode` only if `ops->set_mode` returned 0 | `Phy::set_mode` |
| `of_xlate` post: returned `phy *` belongs to the calling provider OR is ERR_PTR | `PhyProvider::of_xlate` |

### Layer 4: Verus/Creusot functional

Across DT-driven probe: consumer `phy_get` blocks (returns `-EPROBE_DEFER`) until provider's `of_phy_provider_register` runs; once provider list contains the matching node, consumer probe rerun returns a live `phy *` and driver `phy_init` executes. Encoded as a Verus precondition that `phy_get` from consumer A returns Ok iff provider B has registered + `of_xlate(args_A)` returns Ok.

## Hardening

(Generic device-class hardening from `drivers/00-overview.md` applies.)

PHY-subsystem-specific reinforcement:

- **`phy_provider_mutex` covers list traversal + `of_xlate` call** — defense against UAF if provider unregisters mid-lookup.
- **`try_module_get(phy_provider->owner)` before `of_xlate`** — defense against module unload during lookup.
- **`try_module_get(phy->ops->owner)` after lookup** — defense against vendor-driver unload while consumer holds a reference.
- **`of_node_get(children)` held by provider** — DT node lifetime tied to provider lifetime.
- **`init_count` / `power_count` are guarded `int` protected by `phy->mutex`** — but the audit-recommended migration to `refcount_t` is the Rookery `PhyRef::counter` baseline.
- **Per-PHY `lock_class_key`** — defense against false-positive lockdep cycles across shared mutex use by multiple consumers.
- **Regulator enable failure rolls back** — `phy_power_on` undoes `regulator_enable` if `pm_runtime_get_sync` or `ops->power_on` fails.
- **`mode` only persisted on `ops->set_mode` success** — defense against stale `attrs.mode` after a failed reconfigure.
- **`of_xlate` validates `args.args_count`** — Rookery wrapper rejects DT entries with cell-count outside the provider's declared `#phy-cells`.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `phy`, `phy_provider`, `phy_lookup`, and per-driver private state; no userland buffer ever traverses the PHY core directly.
- **PAX_KERNEXEC** — PHY core (`phy-core.c`) lives in W^X kernel text; `phy_ops` vtables, `of_xlate` thunks, and the global lookup helpers all reside in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `phy_init`, `phy_power_on`, `phy_set_mode_ext`, `_of_phy_get`, and the per-driver init/power_on entries that may sleep on PLL lock.
- **PAX_REFCOUNT** — migrate `init_count` and `power_count` to saturating `refcount_t`; per-driver private refcounts (Qualcomm QMP lane share, MediaTek T-PHY tile share) audited under the same rule.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `struct phy`, `phy_provider`, `phy_lookup`, `phy_configure_opts` payloads, and any per-driver context allocated via `devm_*` so stale lane/PLL state cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every syscall path that ultimately touches PHY consumer drivers (USB ioctl, DRM modeset, PCIe sysfs); reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `phy_ops` (`init`, `exit`, `power_on`, `power_off`, `set_mode`, `set_media`, `set_speed`, `configure`, `validate`, `reset`, `calibrate`, `connect`, `disconnect`, `notify_phystate`, `release`) and `phy_provider->of_xlate` marked `__ro_after_init` with kCFI-typed indirect dispatch; vendor drivers MUST declare prototypes via the kCFI-safe macros.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-PHY register-base disclosure (debugfs `phy/phy-N/registers` where present) behind CAP_SYSLOG; suppress `%p` in PLL-lock and lane-trim tracepoints.
- **GRKERNSEC_DMESG** — restrict PHY init-fail, regulator-fail, and PLL-lock-timeout banners to CAP_SYSLOG so attackers cannot probe analog characteristics via dmesg.
- **of_xlate args bounded** — Rookery wrapper rejects DT entries whose `args_count` exceeds the provider's `#phy-cells`; vendor xlate is given a typed argument view with per-cell range checks.
- **init/power refcount pairing** — `Phy::init` + `Phy::exit` and `Phy::power_on` + `Phy::power_off` enforced as scope-paired guards in Rookery (`PhyInitGuard`, `PhyPowerGuard`) so a missing exit is a compile-time error.
- **CAP_SYS_ADMIN strict** — debugfs (`phy/`) and any vendor-specific sysfs control (`/sys/class/phy/phy-N/control`) require CAP_SYS_ADMIN; live mode-switching from userspace gated.
- **Regulator failure rollback audited** — `Phy::power_on` rollback path verified to disable regulator on every error edge; defense against partial power-on leaving rails active.
- **Per-driver `lock_class_key`** — each PHY gets its own lockdep key registered at `phy_create` and unregistered at `phy_release`; defense against lockdep confusion when many consumers share lanes.
- **PLL-lock timeout bounded** — vendor `ops->init` / `ops->power_on` PLL-wait loops capped (typically 200 ms); refuse to advance lifecycle past a stuck PLL; defense against indefinite kernel-mutex hold when analog hangs.

Rationale: a PHY is a privileged piece of analog plumbing — wrong PLL config can damage the device, wrong mode-switch can expose data on the wire, and a stuck refcount can leave a USB/PCIe link live across driver unload. kCFI on `phy_ops`, scoped init/power guards, of_xlate argument bounding, and CAP_SYS_ADMIN on the control surfaces collapse the attack surface from "any DT bug propagates to live silicon" into a verified provider/consumer contract.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-vendor PHY driver internals (Qualcomm QMP, Rockchip Type-C, Samsung USB DRD, MediaTek T-PHY, ...): future Tier-3 docs `phy-qcom-qmp.md`, `phy-rockchip-typec.md`, etc.
- MIPI D-PHY common props (`phy-core-mipi-dphy.c`): future `phy-mipi-dphy.md` Tier-3.
- USB-only legacy PHY (`drivers/usb/phy/`): tracked separately under `usb/00-overview.md`.
- Network PHY (`drivers/net/phy/`): separate subsystem under `net/phy/00-overview.md`.
- Implementation code.
