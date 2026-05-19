# Tier-3: drivers/pinctrl/ — pin control subsystem overview

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pinctrl/00-overview.md
upstream-paths:
  - drivers/pinctrl/core.c
  - drivers/pinctrl/core.h
  - drivers/pinctrl/devicetree.c
  - drivers/pinctrl/pinmux.c
  - drivers/pinctrl/pinconf.c
  - drivers/pinctrl/pinconf-generic.c
  - drivers/pinctrl/pinctrl-generic.c
  - include/linux/pinctrl/pinctrl.h
  - include/linux/pinctrl/pinmux.h
  - include/linux/pinctrl/pinconf.h
  - include/linux/pinctrl/consumer.h
  - include/linux/pinctrl/machine.h
-->

## Summary

The pin control subsystem is the kernel arbiter for SoC pinmuxing (selecting which on-die function — UART, I2C, SPI, GPIO, audio, eMMC, etc. — drives a physical pad) and pin configuration (pull-up/pull-down, drive strength, slew rate, schmitt-trigger, input-enable, debounce, open-drain). A pin controller exposes a fixed pool of numbered pins, groups them into named pingroups, advertises functions that can be muxed onto groups, and accepts per-pin configuration values. Consumer drivers either request named pinctrl states (`default`, `sleep`, `idle`, ...) declared in DT / ACPI / board files, or use the GPIO-back-channel that translates `gpio_request` into a pinmux switch to the GPIO function.

This Tier-3 covers the subsystem shape: per-controller `pinctrl_dev` registration, the three operation vtables (`pinctrl_ops`, `pinmux_ops`, `pinconf_ops`), generic helpers (`pinctrl-generic.c`, `pinconf-generic.c`) for vendor drivers that adopt the boilerplate, and how DT mapping is parsed (cross-ref `pinctrl-core.md`). Per-vendor drivers (cross-ref `pinctrl-amd.md` and the dozens of `pinctrl-*.c` SoC families) layer on this scaffold.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pinctrl_dev` | per-pin-controller class device | `drivers::pinctrl::Controller` |
| `struct pinctrl_desc` | static descriptor: name, pins, ops vtables | `drivers::pinctrl::Descriptor` |
| `struct pinctrl_pin_desc` | per-pin {number, name, drv_data} | `drivers::pinctrl::PinDesc` |
| `struct pingroup` | named array of pin numbers | `drivers::pinctrl::Group` |
| `struct pinfunction` | named array of group names | `drivers::pinctrl::Function` |
| `struct pinctrl_ops` | get_groups_count / get_group_name / get_group_pins / dt_node_to_map | `Controller::pctlops` |
| `struct pinmux_ops` | request / free / get_functions_count / get_function_name / get_function_groups / set_mux / gpio_request_enable / gpio_disable_free / gpio_set_direction / strict | `Controller::pmxops` |
| `struct pinconf_ops` | pin_config_get / _set / _group_get / _group_set / dbg_show | `Controller::confops` |
| `pinctrl_register_and_init(desc, dev, drvdata, &pctldev)` / `pinctrl_enable(pctldev)` | two-phase register: alloc + init then enable | `Controller::register_and_init` / `_enable` |
| `devm_pinctrl_register_and_init` / `devm_pinctrl_register` | devres-managed register | `Controller::devm_register` |
| `pinctrl_unregister(pctldev)` | drop controller from global list | `Controller::unregister` |
| `pinctrl_add_gpio_range(pctldev, range)` / `_remove_gpio_range` | declare GPIO-number → pin-number mapping for a gpio_chip | `Controller::add_gpio_range` |
| `pinctrl_generic_*` (group + function helpers in `pinctrl-generic.c`) | radix-tree backed group/function store for vendor drivers | `Controller::generic_*` |
| `pinconf_generic_*` (in `pinconf-generic.c`) | dt-parsing helpers for `bias-pull-up`, `drive-strength`, etc. | `Controller::pinconf_generic_*` |

## Compatibility contract

REQ-1: Per-controller `pinctrl_desc.pins[]` is a contiguous-ish set of {number, name} descriptors; pin numbers are private to that controller (no global namespace) and may be sparse but bounded by `desc->npins`.

REQ-2: A controller MUST supply `pctlops` (with at least `get_groups_count`, `get_group_name`, `get_group_pins`); MAY supply `pmxops` if it supports muxing; MAY supply `confops` if it supports per-pin configuration.

REQ-3: `pinmux_ops.strict = true` forbids simultaneous use of a pin as GPIO and as a muxed function (cross-ref `pinmux.c` `pin_request`).

REQ-4: GPIO consumers reach pinmux via `pinctrl_gpio_request()`; the core walks registered gpio_ranges, locates the owning controller, and calls `pmxops->gpio_request_enable`.

REQ-5: DT mapping: `pinctrl_ops.dt_node_to_map` parses a DT pinconf node into `pinctrl_map[]` entries (MUX_GROUP / CONFIGS_PIN / CONFIGS_GROUP / DUMMY_STATE); `dt_free_map` releases them.

REQ-6: Per-consumer device, `dev->pins` is auto-bound in `driver_probe_device` to `default` then `init` states (in that order) before `probe` runs.

REQ-7: Generic pin config (CONFIG_GENERIC_PINCONF) encodes 32 standard parameters in `unsigned long` via `PIN_CONF_PACKED(param, arg)` so drivers can share parsing.

REQ-8: Per-pin `pin_desc` carries `mux_usecount`, `mux_owner`, `gpio_owner`, and a per-pin mutex — the core arbitrates conflicting requests across consumers.

REQ-9: debugfs (`<debugfs>/pinctrl/`) exposes per-controller pin/group/function tables and the current mux/conf state behind CONFIG_DEBUG_FS.

REQ-10: A `pinctrl_dummy_state` mode lets boards without a real pinctrl provider still satisfy `pinctrl_get_default()` for shared drivers; opt-in via `pinctrl_provide_dummies()`.

## Acceptance Criteria

- [ ] AC-1: A boot on AMD KERNCZ + on a DT SoC (e.g. Rockchip) registers per-controller `pinctrl_dev` entries visible in `<debugfs>/pinctrl/pinctrl-devices`.
- [ ] AC-2: `pinctrl-test` (or kselftest equivalent) registers a synthetic controller, requests default + sleep states, and observes set_mux + pin_config_set ordering matching upstream.
- [ ] AC-3: GPIO request on a strict pin controller refuses if pin already mux-owned and vice-versa.
- [ ] AC-4: DT-driven board boot satisfies all consumer `pinctrl-0` references; missing required state fails probe deterministically.
- [ ] AC-5: `<debugfs>/pinctrl/pinctrl-handles` shows per-consumer pinctrl-handle + active state.

## Architecture

```
+----------------------------+   register_and_init   +------------------+
| vendor pinctrl-foo.c       | --------------------> | pinctrl core     |
|  - static pinctrl_desc     |                       |  pinctrldev_list |
|  - pinctrl_ops             |                       |  pinctrl_maps    |
|  - pinmux_ops              |   add_gpio_range      |                  |
|  - pinconf_ops             | --------------------> |  gpio_ranges     |
+----------------------------+                       +------------------+
                                                              |
                          consumer driver probe               | pinctrl_get
                          ------------------------------------v
                          struct pinctrl { states, dt_maps, kref }
                                |
                                | pinctrl_select_state(state)
                                v
                          per-setting MUX_GROUP -> pmxops->set_mux
                          per-setting CONFIGS_* -> confops->pin_config_set
```

The core keeps three global lists: `pinctrldev_list` (every registered controller), `pinctrl_list` (every per-consumer `pinctrl` handle), `pinctrl_maps` (machine + DT mapping table). Each list is guarded by its own mutex; per-controller mutations hold `pctldev->mutex`.

Vendor drivers fall into two flavours:
- **Generic**: use `pinctrl_generic_*` to insert groups into `pctldev->pin_group_tree` (radix tree) and `pinctrl_generic_*` analogue for functions. Boilerplate is shared (see `pinctrl-amd.c`).
- **Bespoke**: implement their own group/function tables and supply hand-written `get_groups_count` / `get_group_name` / `get_group_pins` callbacks (older drivers).

## Hardening

- Two-phase register (`pinctrl_register_and_init` + `pinctrl_enable`) lets a driver fail before the controller becomes globally visible.
- Per-pin mux_usecount + gpio_owner enforce strict-mode conflict detection.
- DT map parsing always copies strings via `kstrdup_const` so the consumer DT-node lifetime doesn't bleed into the kept map.
- `pinctrl_dummy_state` is opt-in; the core never silently fabricates a default-state success.
- All vtable callbacks dereferenced from `pctldev->desc->{pctlops,pmxops,confops}` which is `__ro_after_init` once `pinctrl_enable` returns.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `pinctrl`, `pinctrl_state`, `pinctrl_setting`, `pin_desc`, `pinctrl_maps`; debugfs reads bounded with explicit length checks; group / function identifier copy-out to userspace strictly length-checked.
- **PAX_KERNEXEC** — pinctrl core text W^X; `pinctrl_ops`, `pinmux_ops`, `pinconf_ops` dispatch and the dt_node_to_map path live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel stack offset across `pinctrl_select_state`, `dt_node_to_map`, `pinmux_request_pin`, and GPIO-request fast paths.
- **PAX_REFCOUNT** — saturating `refcount_t` / `kref` on `struct pinctrl` consumer handles and `pinctrl_dev->desc->owner` module references; overflow trap defeats handle-leak UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `pinctrl_state`, `pinctrl_setting`, dt_map allocations, and per-pin `mux_owner` / `gpio_owner` string buffers so stale ownership cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every debugfs / sysfs entry into pinctrl; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `pinctrl_ops`, `pinmux_ops`, `pinconf_ops` vtables marked `__ro_after_init` with kCFI-typed indirect dispatch; rejects mis-typed `set_mux` / `pin_config_set` calls injected via corrupted `pinctrl_desc`.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of vendor pinctrl callbacks and per-controller register-base pointers behind CAP_SYSLOG; suppress `%p` in pinctrl tracepoints.
- **GRKERNSEC_DMESG** — restrict pinctrl probe / map-parse / strict-mode-conflict banners to CAP_SYSLOG so attackers cannot probe pad layout via dmesg.
- **debugfs CAP_SYS_ADMIN** — `<debugfs>/pinctrl/*` (pinmux-pins, pinmux-functions, pinconf-pins, pinctrl-handles, pinctrl-maps) require CAP_SYS_ADMIN; refuse mode-bit relaxation.
- **group / function id PAX_USERCOPY** — every group_selector / function_selector copy-out (debugfs, ioctl extensions, vendor sysfs) length-checked against `pctldev->num_groups` / `num_functions`.
- **pinctrl_ops kCFI** — every indirect dispatch through pctlops / pmxops / confops type-checked; refuse vendor drivers that mismatch the canonical signatures.
- **gpio-range overlap reject** — `pinctrl_add_gpio_range` refuses overlapping ranges across controllers, defending against confused-deputy gpio→pinmux routing.
- **Strict-mode enforcement** — `pinmux_ops.strict` validated at register time; controllers claiming strict must implement both `gpio_request_enable` and conflict detection.

Rationale: pinctrl is the arbiter of which silicon function drives which physical pad, including pads carrying secrets (eFuse strap, JTAG, secure-debug). A corrupted `pmxops->set_mux` dispatch, a missed strict-mode check, or a debugfs write that bypasses CAP_SYS_ADMIN can route arbitrary internal signals to externally accessible pads. RAP/kCFI on ops vtables, CAP_SYS_ADMIN on debugfs, refcount-overflow trapping, and strict-mode enforcement turn the pin controller from a convenience layer into a structural pad-routing boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `group_selector_no_oob` | OOB | every `get_group_name(sel)` and `get_group_pins(sel)` call sees `sel < pctldev->num_groups`. |
| `function_selector_no_oob` | OOB | every `set_mux(func, group)` call sees `func < num_functions` and `group < num_groups`. |
| `pinctrl_handle_no_uaf` | UAF | `kref` on `struct pinctrl` keeps it alive across all `dt_maps` references; free deferred to last `pinctrl_put`. |

### Layer 2: TLA+

`models/pinctrl/select_state.tla`: proves `pinctrl_select_state(s)` is atomic w.r.t. concurrent consumer requests on the same handle — `state` field transitions monotonically through `settings` list without observable partial application.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `pinctrl_register_and_init` post: `pctldev` visible in `pinctrldev_list` iff `desc->pctlops` non-null and required callbacks non-null | `Controller::register_and_init` |
| `pinmux_request_pin` post: pin `mux_usecount > 0 ⇒ mux_owner != NULL` | `Controller::pinmux_request_pin` |
| Strict-mode invariant: `pmxops.strict ⇒ (mux_usecount > 0) XOR (gpio_owner != NULL)` per pin | strict-mode controllers |

### Layer 4: Verus/Creusot functional

Consumer probe → `pinctrl_get(dev)` → `pinctrl_select_state(default)` → per-setting `set_mux` + `pin_config_set` complete → consumer can drive pad → consumer remove → `pinctrl_put` → all `mux_usecount` decrement to zero. Encoded as Verus invariant chained with `pinctrl-core.md`.

## Out of Scope

- Per-vendor SoC details (covered in `pinctrl-amd.md`, future per-vendor docs)
- Intel pinctrl (covered in future `pinctrl-intel.md`)
- ACPI pinctrl bindings (covered in future `pinctrl-acpi.md`)
- 32-bit-only paths
- Implementation code
