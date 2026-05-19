# Tier-3: drivers/pinctrl/{core,devicetree,pinmux,pinconf}.c — pinctrl core mechanics

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pinctrl/00-overview.md
upstream-paths:
  - drivers/pinctrl/core.c
  - drivers/pinctrl/core.h
  - drivers/pinctrl/devicetree.c
  - drivers/pinctrl/devicetree.h
  - drivers/pinctrl/pinmux.c
  - drivers/pinctrl/pinmux.h
  - drivers/pinctrl/pinconf.c
  - drivers/pinctrl/pinconf.h
-->

## Summary

Core mechanics of the pinctrl subsystem: how a `pinctrl_dev` is registered (`core.c`, ~2400 LOC), how the per-consumer `struct pinctrl` handle is built (state lookup, ref counting), how DT mapping is parsed into `pinctrl_map[]` chunks (`devicetree.c`, ~430 LOC), how a mux request is arbitrated (`pinmux.c`, ~1000 LOC: pin-request, set-mux, free), and how pinconf settings are applied (`pinconf.c`, ~380 LOC: validation + dispatch to `pinconf_ops->pin_config_{get,set}`).

This Tier-3 narrows in on the four core C files: `core.c` (registration, gpio-range registry, per-consumer pinctrl alloc/free, state selection), `devicetree.c` (DT node → map list), `pinmux.c` (`pin_request` / `set_mux` / `pin_free` plumbing), `pinconf.c` (pinconf dispatch and debugfs).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `pinctrl_register_and_init(desc, dev, drvdata, &pctldev)` | alloc + sanity-check pctldev, register pin descriptors in radix tree | `Core::register_and_init` |
| `pinctrl_enable(pctldev)` | publish pctldev to `pinctrldev_list`, hog `default`/`sleep` states | `Core::enable` |
| `pinctrl_unregister(pctldev)` | remove from list, drop hogs | `Core::unregister` |
| `pinctrl_get(dev)` / `pinctrl_put(p)` | per-consumer handle alloc / kref drop | `Consumer::get` / `_put` |
| `pinctrl_lookup_state(p, name)` | find named state in `p->states` | `Consumer::lookup_state` |
| `pinctrl_select_state(p, state)` | apply each setting (mux, conf) in order | `Consumer::select_state` |
| `create_pinctrl(dev, pctldev)` | walk `pinctrl_maps`, build per-consumer settings list | `Consumer::create_pinctrl` |
| `add_setting(p, pctldev, map)` | translate map entry to setting in correct state | `Consumer::add_setting` |
| `pinctrl_register_mappings(maps, num)` / `_unregister_mappings(maps)` | static board / machine mapping table | `Core::register_mappings` |
| `pinctrl_dt_to_map(p, pctldev)` | walk consumer DT node for `pinctrl-N` phandles | `Core::dt_to_map` |
| `dt_remember_or_free_map(p, state, pctldev, map, num)` | track DT-allocated map for later free | `Core::dt_remember_or_free_map` |
| `pinmux_check_ops(pctldev)` | validate `pmxops` has required callbacks | `Core::pinmux_check_ops` |
| `pin_request(pctldev, pin, owner, range)` | claim pin, enforce strict mode | `Core::pin_request` |
| `pin_free(pctldev, pin, range)` | release pin | `Core::pin_free` |
| `pinmux_enable_setting(setting)` / `_disable_setting(setting)` | request pins, dispatch `set_mux` | `Core::pinmux_enable_setting` |
| `pinconf_check_ops(pctldev)` | validate `confops` has required callbacks | `Core::pinconf_check_ops` |
| `pin_config_set(devname, name, config)` / `pin_config_get` | one-shot pinconf API | `Core::pin_config_set` |
| `pinconf_apply_setting(setting)` | dispatch `pin_config_set` over `setting.data.configs` | `Core::pinconf_apply_setting` |
| `pinctrl_add_gpio_range(pctldev, range)` | declare GPIO→pin mapping | `Core::add_gpio_range` |
| `pinctrl_find_gpio_range_from_pin_nolock(pctldev, pin)` | reverse lookup | `Core::find_gpio_range_from_pin` |

## Compatibility contract

REQ-1: `pinctrl_register_and_init` MUST reject `desc` with NULL `pctlops`, NULL `pctlops->get_groups_count`, NULL `name`, or zero `npins`.

REQ-2: Pin descriptors are stored in a radix tree (`pctldev->pin_desc_tree`) keyed by pin number; lookup via `pin_desc_get(pctldev, pin)` returns NULL for unknown pins (never traps).

REQ-3: Per-consumer handle creation walks `pinctrl_maps` (machine table) + parses consumer DT node; both contribute settings indexed by state name; `pinctrl_lookup_state` returns -ENODEV if state absent (caller chooses error policy).

REQ-4: `pinctrl_select_state` is idempotent w.r.t. an already-selected state — early returns without re-driving hardware.

REQ-5: A setting of type `PIN_MAP_TYPE_MUX_GROUP` triggers `pin_request` on every pin in the group, then `pmxops->set_mux(func, group)`; on any failure, partial requests are rolled back.

REQ-6: A setting of type `PIN_MAP_TYPE_CONFIGS_{PIN,GROUP}` iterates over the configs array, dispatching `confops->pin_config_set` (or `_group_set`) per element.

REQ-7: DT mapping: `pinctrl-N` phandle list per consumer node parsed by `pinctrl_dt_to_map`; each phandle's pinconf node is handed to the target pctldev's `dt_node_to_map`; emitted maps are remembered on `p->dt_maps` for `pinctrl_dt_free_maps` at consumer remove.

REQ-8: `pinmux_ops.strict == true` MUST make `pin_request` reject claims when `mux_usecount > 0` OR `gpio_owner != NULL` already set.

REQ-9: `pinctrl_add_gpio_range` MUST refuse a range that overlaps an existing range on the same controller (defensive; cross-ref `pinctrl/00-overview.md` REQ-13).

REQ-10: `pinctrl_force_default` / `_force_sleep` API is callable from suspend paths and applies the hogged states without taking sleeping locks.

## Acceptance Criteria

- [ ] AC-1: Synthetic controller register → handle-get → state-select → state-put leaves all per-pin `mux_usecount == 0`.
- [ ] AC-2: Strict-mode controller: GPIO request after mux claim returns -EBUSY; mux claim after GPIO request returns -EBUSY.
- [ ] AC-3: DT consumer parse with malformed `pinctrl-N` phandle returns -EINVAL and does not leak map memory.
- [ ] AC-4: `pinctrl_register_mappings` followed by `pinctrl_unregister_mappings` leaves `pinctrl_maps` empty.
- [ ] AC-5: `pinctrl_select_state(p, p->state)` (same-state reselect) does not call `set_mux` or `pin_config_set`.

## Architecture

`Core` lives in `drivers::pinctrl::core`:

```
struct Controller {
  desc: &'static Descriptor,
  pin_desc_tree: RadixTree<u32, PinDesc>,
  pin_group_tree: Option<RadixTree<u32, GroupDesc>>,
  pin_function_tree: Option<RadixTree<u32, FunctionDesc>>,
  num_groups: u32,
  num_functions: u32,
  gpio_ranges: Mutex<List<GpioRange>>,
  dev: &Device,
  owner: &Module,
  driver_data: *mut u8,
  p: Option<Arc<Consumer>>,             // self-pinctrl handle for hogs
  hog_default: Option<Arc<PinctrlState>>,
  hog_sleep: Option<Arc<PinctrlState>>,
  mutex: Mutex<()>,
  device_root: Option<DentryRef>,       // debugfs
}

struct Consumer {
  dev: &Device,
  states: List<PinctrlState>,
  state: Option<&PinctrlState>,
  dt_maps: List<DtMap>,
  users: Refcount,
}

struct PinctrlState {
  name: KString,
  settings: List<Setting>,
}

enum Setting {
  Mux { pctldev: &Controller, group: u32, func: u32 },
  ConfigsPin { pctldev: &Controller, pin: u32, configs: KVec<u64> },
  ConfigsGroup { pctldev: &Controller, group: u32, configs: KVec<u64> },
  Dummy,
}

struct PinDesc {
  pctldev: &Controller,
  name: KStr,
  dynamic_name: bool,
  drv_data: *mut u8,
  mux_usecount: u32,           // PINMUX
  mux_owner: Option<KString>,  // PINMUX
  mux_setting: Option<&Setting>,
  gpio_owner: Option<KString>,
  mux_lock: Mutex<()>,
}
```

Registration `Core::register_and_init`:
1. Validate `desc` (name + npins + pctlops + required callbacks).
2. `kzalloc(pctldev)`; copy `desc` pointer; init pin_desc_tree (radix).
3. For each `desc->pins[i]`: alloc `pin_desc`, insert in radix at `pin.number`.
4. If `pmxops` non-NULL: `pinmux_check_ops`; init per-pin `mux_lock`.
5. If `confops` non-NULL: `pinconf_check_ops`.
6. If `pmxops->strict`: validate `gpio_request_enable` non-NULL.
7. Return pctldev with `pinctrldev_list` NOT yet populated (publish on `pinctrl_enable`).

Enable `Core::enable`:
1. `mutex_lock(pinctrldev_list_mutex)`.
2. `list_add_tail(&pctldev->node, &pinctrldev_list)`.
3. Hog states: `pinctrl_get(pctldev->dev)` → `lookup_state("default")` / `lookup_state("sleep")` and call `pinctrl_select_state(default)`.
4. Register debugfs entries.

Consumer handle build `Consumer::create_pinctrl`:
1. `kzalloc(p)`; `kref_init(&p->users)`.
2. Walk `pinctrl_maps`: for every map referencing `dev_name(dev)`, `add_setting`.
3. Parse consumer DT: `pinctrl_dt_to_map(p, pctldev)` walks `pinctrl-N` phandle list, calls per-pctldev `dt_node_to_map`, remembers on `p->dt_maps`.
4. Insert each setting into per-state list; if state absent, create it.
5. Bound: `list_add_tail(&p->node, &pinctrl_list)`.

State selection `Consumer::select_state(state)`:
1. If `p->state == state`: early return 0.
2. For each setting in current `p->state`: `pinmux_disable_setting` / `pinconf` no-op.
3. For each setting in new `state`: `pinmux_enable_setting` or `pinconf_apply_setting`.
4. On failure: roll back applied settings; restore `p->state`.

Pin request `Core::pin_request(pctldev, pin, owner, range)`:
1. `pin_desc_get(pctldev, pin)` → `pd`; -EINVAL if NULL.
2. `mutex_lock(&pd->mux_lock)`.
3. If `range` (GPIO request): refuse if `pd->mux_usecount > 0 && pmxops->strict`; set `gpio_owner`.
4. Else (mux request): refuse if `pd->gpio_owner != NULL && pmxops->strict`; increment `mux_usecount`; set `mux_owner`.
5. `pmxops->request(pctldev, pin)` or `pmxops->gpio_request_enable(pctldev, range, pin)`.
6. Rollback on driver-callback failure.

DT map parse `Core::dt_to_map(p, pctldev)`:
1. Walk consumer DT-node for `pinctrl-N` and `pinctrl-names-N` properties.
2. For each phandle: lookup target pctldev (`get_pinctrl_dev_from_of_node`).
3. Call target `pctlops->dt_node_to_map(pctldev, np, &map, &num_maps)`.
4. `dt_remember_or_free_map(p, statename, pctldev, map, num_maps)` — fill `dev_name` strings via `kstrdup_const`.
5. On consumer free: `pinctrl_dt_free_maps(p)` walks `p->dt_maps`, calls `pctldev->pctlops->dt_free_map`, releases.

## Hardening

- Per-pin `mux_lock` serializes the request/free path; no cross-pin lock-order issues.
- `pinctrl_dummy_state` opt-in only; never silently fabricated.
- `pin_desc_tree` lookup returns NULL for unknown pin numbers — never OOB array access.
- DT map strings always `kstrdup_const` copied to keep consumer-DT-node free decoupled from map free.
- Setting rollback on partial-state-apply restores `p->state` so consumer sees a consistent state.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `pin_desc`, `pinctrl`, `pinctrl_state`, `pinctrl_setting`, `pinctrl_dt_map`, and copies of DT map entries; debugfs reads of pin/group/function tables length-bounded.
- **PAX_KERNEXEC** — `core.c`, `devicetree.c`, `pinmux.c`, `pinconf.c` text W^X; vtable dispatch out of `__ro_after_init` cells.
- **PAX_RANDKSTACK** — randomize kernel stack offset across `pinctrl_select_state`, `pin_request`, `pinmux_enable_setting`, `pinconf_apply_setting`, and `pinctrl_dt_to_map`.
- **PAX_REFCOUNT** — saturating `kref` on `struct pinctrl` consumer handles; saturating `refcount_t` on per-pin `mux_usecount`; overflow trap defeats handle-leak / use-count overflow UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `pin_desc`, `pinctrl_state`, `pinctrl_setting`, `pinctrl_dt_map`, and `mux_owner` / `gpio_owner` string allocations so stale ownership cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every debugfs entry into `core.c` / `pinmux.c` / `pinconf.c`; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `pinctrl_ops`, `pinmux_ops`, `pinconf_ops` indirect dispatch kCFI-typed; refuses corrupted `pctldev->desc` pointing at wrong signature.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `pinmux_enable_setting`, `pin_request`, `pinconf_apply_setting` symbols behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict pinctrl probe / map-parse / strict-conflict / pinconf-apply banners to CAP_SYSLOG so attackers cannot probe board pad layout via dmesg.
- **Pin descriptor PAX_REFCOUNT** — `mux_usecount` is saturating; overflow saturates and traps rather than wrap.
- **Default state lookup bounded** — `pinctrl_lookup_state` walks `p->states` with a hard list-length bound; refuses pathological state-list constructions from corrupted maps.
- **gpio-range overlap reject** — `pinctrl_add_gpio_range` walks existing ranges and refuses any overlap (pin-base + npins), per-controller, before insertion.
- **DT map name length cap** — `dev_name` strings entering `dt_remember_or_free_map` capped at `PATH_MAX/4`; refuse pathological DT-node names.
- **debugfs CAP_SYS_ADMIN** — `<debugfs>/pinctrl/{pinctrl-handles,pinctrl-maps,pinmux-pins,pinconf-pins}` require CAP_SYS_ADMIN to read.

Rationale: the core is the only place where pin-ownership and mux-state are arbitrated. A corrupted `pin_request` (missed strict check, `mux_usecount` overflow), a forged DT map (string lifetime bug), or a debugfs write that bypasses CAP_SYS_ADMIN can silently route secure-only pads to externally accessible functions. Refcount saturation, gpio-range overlap rejection, kCFI on vtable dispatch, and CAP_SYS_ADMIN gating turn pad routing into a structural enforcement boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pin_request_no_overflow` | OVERFLOW | `mux_usecount` saturates on overflow; never wraps. |
| `state_select_no_partial` | INVARIANT | partial-apply failure restores prior `p->state` and undoes already-applied settings. |
| `dt_map_no_uaf` | UAF | `pinctrl_dt_map.dev_name` lifetime owned by map; DT-node free does not invalidate. |
| `gpio_range_no_overlap` | INVARIANT | `pinctrl_add_gpio_range` rejects overlap; existing ranges remain consistent. |

### Layer 2: TLA+

`models/pinctrl/strict_mode.tla`: proves that under `pmxops.strict`, no interleaving of concurrent `pin_request(GPIO)` and `pin_request(MUX)` admits both succeeding on the same pin.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `pin_request` post: `pd->mux_usecount` incremented iff non-strict OR (`gpio_owner == NULL`) | `Core::pin_request` |
| `pinmux_enable_setting` post: every pin in `setting.group` has `mux_owner == setting.dev_name` | `Core::pinmux_enable_setting` |
| `select_state` post: `p->state == new_state` iff every setting applied successfully | `Consumer::select_state` |

### Layer 4: Verus/Creusot functional

`pinctrl_get(dev) → select_state(default) → select_state(sleep) → select_state(default) → put` leaves all `mux_usecount == 0` after `put`. Encoded as Verus invariant chained with `pinctrl/00-overview.md`.

## Out of Scope

- Per-vendor `dt_node_to_map` parsers (covered in vendor docs)
- ACPI `_PRT` / `_DSD` pinctrl bindings (future)
- pinconf-generic.c parameter helpers (future companion doc)
- pinctrl-generic.c group/function helpers (future companion doc)
- 32-bit-only paths
- Implementation code
