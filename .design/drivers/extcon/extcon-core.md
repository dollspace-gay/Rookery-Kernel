# Tier-3: drivers/extcon/extcon.c — external connector framework (cable detection + notifier chain + properties)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/extcon/extcon.c
  - drivers/extcon/extcon.h
  - drivers/extcon/devres.c
  - include/linux/extcon.h
  - include/linux/extcon-provider.h
-->

## Summary

The `extcon` (external connector) framework abstracts cable / accessory detection on user-facing physical connectors — USB Type-A / Type-C / micro-USB OTG, HDMI, headphone jacks, dock connectors, charger types — so consumer drivers (USB role switch, charger manager, audio codec) react to "cable attached / detached" without polling per-chip detector registers.

A detector driver (e.g. PMIC USB-VBUS comparator, type-C PHY, GPIO-jack detector) registers a `extcon_dev` advertising a `supported_cable[]` array of `EXTCON_*` IDs (USB, USB_HOST, CHG_USB_SDP, CHG_USB_CDP, CHG_USB_DCP, JACK_HEADPHONE, DISP_HDMI, USB_VBUS, ...). On state change the detector calls `extcon_set_state_sync(edev, EXTCON_USB, true)`; the framework updates per-cable state, broadcasts a uevent (`CABLE_NAME=USB STATE=1`), and walks the per-cable + per-device notifier chains to wake consumer subscribers.

This Tier-3 covers `drivers/extcon/extcon.c` (~1488 lines: framework + per-cable state machine + sysfs + uevent + notifier-chain + properties + device-tree consumer lookup), `extcon.h` (private internals), and `devres.c` (managed variants).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct extcon_dev` | per-detector device | `drivers::extcon::ExtconDev` |
| `struct extcon_cable` | per-supported-cable instance | `drivers::extcon::Cable` |
| `struct extcon_specific_cable_nb` | per-cable notifier-block + matching info | `drivers::extcon::SpecificCableNb` |
| `extcon_dev_allocate(supported_cable)` / `extcon_dev_free(edev)` | per-driver alloc | `ExtconDev::alloc` / `_free` |
| `extcon_dev_register(edev)` / `extcon_dev_unregister(edev)` | sysfs + class entry add/remove | `ExtconDev::register` |
| `extcon_set_state(edev, id, state)` / `_set_state_sync(edev, id, state)` | per-cable state set; `_sync` adds uevent + notifier broadcast | `ExtconDev::set_state` / `_sync` |
| `extcon_get_state(edev, id)` | per-cable read | `ExtconDev::get_state` |
| `extcon_sync(edev, id)` | re-emit uevent + notifier for a cable (used after batched `set_state`) | `ExtconDev::sync` |
| `extcon_set_property(edev, id, prop, value)` / `_set_property_sync(...)` / `_get_property(...)` | per-cable per-property (e.g. typec polarity, charger current) | `ExtconDev::set_property` |
| `extcon_set_property_capability(edev, id, prop)` / `_get_property_capability(...)` | per-cable advertised property set | `ExtconDev::property_capability` |
| `extcon_register_notifier(edev, id, nb)` / `_unregister_notifier(...)` | per-cable subscriber registration | `ExtconDev::register_notifier` |
| `extcon_register_notifier_all(edev, nb)` / `_unregister_notifier_all(...)` | per-device (any cable) subscriber | `ExtconDev::register_notifier_all` |
| `extcon_get_extcon_dev(name)` | global name lookup | `ExtconDev::get_by_name` |
| `extcon_find_edev_by_node(np)` / `extcon_get_edev_by_phandle(dev, idx)` | DT lookup | `ExtconDev::find_by_node` |
| `devm_extcon_register_notifier(dev, edev, id, nb)` + companion devm_* | devres variants | `devres::*` |

## Compatibility contract

REQ-1: Cable ID space: `EXTCON_NONE..EXTCON_NUM` defined in `include/linux/extcon.h`; covers connectors (USB, USB_HOST, USB_VBUS), charger types (CHG_USB_SDP/CDP/DCP/ACA/FAST/SLOW), jacks (JACK_MICROPHONE, JACK_HEADPHONE, JACK_LINE_IN/OUT, JACK_VIDEO_IN/OUT, JACK_SPDIF_IN/OUT), and displays (DISP_HDMI, DISP_MHL, DISP_DVI, DISP_VGA, DISP_DP). Detector advertises a sentinel-terminated subset.

REQ-2: Per-cable state cached as bit in `edev->state` (`u32`); 32-bit limit ≥ `EXTCON_NUM` enforced at compile time via `BUILD_BUG_ON`.

REQ-3: Per-cable properties: each `EXTCON_PROP_*` (USB_TYPEC_POLARITY, USB_VBUS, USB_SS, CHG_MIN/MAX/USB_CURRENT, JACK_LEFT/RIGHT_AVOL, DISP_HPD, ...) stored in `cable->props[]`; property capability bitmap per-cable in `cable->prop_cap`.

REQ-4: Sysfs class: `/sys/class/extcon/extconN/` with attributes `name`, `state` (all-cables snapshot), per-cable `cable.N/{name,state}`.

REQ-5: Uevent broadcast on `_set_state_sync`: `kobject_uevent_env` with envp `EXTCON_NAME=<edev-name>`, per-cable `CABLE_NAME=<cable>` + `STATE=0|1`, and `EXTCON_STATE=<bitmask>` summary.

REQ-6: Notifier chains: per-cable chain (`cable->nh`) carries `union extcon_property_value` payload + boolean attached state to callback; per-device "all" chain (`edev->nh_all`) carries the same envelope with cable id.

REQ-7: DT consumer lookup: `extcon = <&phandle>` (single) or `extcon-cables` + `extcon-cable-names` (multi); core resolves via `extcon_find_edev_by_node` walking the global `extcon_dev_list`.

REQ-8: `devm_extcon_register_notifier(dev, edev, id, nb)` releases the notifier on device-drop; consumer never leaks subscription.

REQ-9: Atomicity: `extcon_set_state` updates bit + emits notifier under `edev->lock` (mutex); `_set_state_sync` additionally emits the uevent under the same lock.

REQ-10: Mutual-exclusion across cables: per-detector `mutually_exclusive[]` declares "if cable A is asserted, cable B must be deasserted"; framework enforces by clearing the conflicting cable's bit during set.

REQ-11: Per-cable per-property notification: `_set_property_sync` fires per-cable notifier with the property delta; consumers filter via `event->property_value`.

REQ-12: Lifetime: per-extcon-dev refcount via class device kref; `extcon_dev_unregister` blocks if any consumer holds an active notifier subscription.

## Acceptance Criteria

- [ ] AC-1: USB OTG dev board: cable plug detection by PMIC sets `EXTCON_USB`; `cat /sys/class/extcon/extcon0/state` reads "USB=1"; uevent observed by udev.
- [ ] AC-2: USB role-switch driver subscribing via `extcon_register_notifier(edev, EXTCON_USB_HOST, &nb)` toggles role on cable swap.
- [ ] AC-3: Type-C polarity property: `extcon_set_property_sync(edev, EXTCON_USB, EXTCON_PROP_USB_TYPEC_POLARITY, 1)` fires per-cable notifier delivering the value.
- [ ] AC-4: Mutual-exclusion: detector with `mutually_exclusive = { (EXTCON_USB, EXTCON_USB_HOST) }` setting USB clears USB_HOST + emits both uevents.
- [ ] AC-5: Charger manager subscribes to CHG_USB_SDP/CDP/DCP via `register_notifier_all` and selects current limit per advertised type.
- [ ] AC-6: rmmod of detector driver while consumer holds notifier fails until consumer unregisters.
- [ ] AC-7: DT phandle resolution: `extcon = <&pmic_usb>` consumer finds the right edev; missing phandle returns `-EPROBE_DEFER`.
- [ ] AC-8: `extcon_get_property` on unsupported cable returns `-EPERM`; capability-table truthful.

## Architecture

`ExtconDev` + `Cable` layout:

```
struct ExtconDev {
  name: KString,
  supported_cable: Vec<u32>,            // EXTCON_NONE-terminated
  mutually_exclusive: Vec<u32>,         // bitmask pairs
  dev: Device,                          // class extcon, parent = detector
  state: AtomicU32,                     // bit per cable
  id_seed: u32,
  lock: Mutex<()>,
  max_supported: u32,
  cables: Vec<Cable>,                   // per supported cable
  nh_all: BlockingNotifierHead,         // any-cable subscribers
  nh: Vec<BlockingNotifierHead>,        // per-cable subscribers (parallel to cables)
  d_index: u32,                         // class-allocated extconN id
  entry: ListHead,                      // extcon_dev_list global
}

struct Cable {
  edev: &ExtconDev,
  cable_index: u32,                     // index into supported_cable
  attr_g: AttributeGroup,
  attrs: Vec<Attribute>,
  attr_name: DeviceAttribute,
  attr_state: DeviceAttribute,
  prop_cap: [u32; PROP_CAP_WORDS],      // per-property capability bitmap
  usb_propval: [PropertyValue; USB_PROP_COUNT],
  chg_propval: [PropertyValue; CHG_PROP_COUNT],
  jack_propval: [PropertyValue; JACK_PROP_COUNT],
  disp_propval: [PropertyValue; DISP_PROP_COUNT],
}
```

Registration `extcon_dev_register(edev)`:
1. Validate `supported_cable` is non-empty + EXTCON_NONE-terminated.
2. Compute `max_supported` (count).
3. Allocate per-cable `cables[]` + per-cable `nh[]` arrays.
4. Allocate sysfs attributes (`name`, `state`, per-cable `cable.N/{name,state}`).
5. `ida_alloc(&extcon_dev_ida, ...)` → `d_index`; `dev_set_name("extcon%u", d_index)`.
6. `device_register(&edev->dev)`; add to `extcon_dev_list`.

State change `extcon_set_state_sync(edev, id, state)`:
1. Validate `id` in `supported_cable`; resolve cable index.
2. `mutex_lock(&edev->lock)`.
3. Compute new `state` bitmask: set/clear bit; apply mutual-exclusion (clear conflicting bits).
4. If unchanged: unlock, return 0 (no spurious uevent).
5. Atomic-store new `edev->state`.
6. For each changed cable bit:
   - Blocking-notifier-call on `edev->nh[index]` with `extcon_event { value, state }`.
   - Blocking-notifier-call on `edev->nh_all` with `extcon_event { id, state }`.
7. `kobject_uevent_env(&edev->dev.kobj, KOBJ_CHANGE, envp)` with `CABLE_NAME=...&STATE=...`.
8. Unlock.

Property set `extcon_set_property_sync(edev, id, prop, val)`:
1. Validate `id` supported + `prop` capable.
2. `mutex_lock`.
3. Cache new value in `cable->{usb,chg,jack,disp}_propval[prop]`.
4. Blocking-notifier-call delivering `extcon_event { property, value }`.
5. Unlock.

Consumer registration `extcon_register_notifier(edev, id, nb)`:
1. Resolve cable index from `id`.
2. `blocking_notifier_chain_register(&edev->nh[index], nb)`.
3. Bump module refcount on detector.

DT lookup `extcon_get_edev_by_phandle(dev, idx)`:
1. `of_parse_phandle(dev->of_node, "extcon", idx)` → np.
2. Walk `extcon_dev_list` matching `dev_of_node(edev->dev.parent) == np`.
3. Return `&edev` or `-EPROBE_DEFER` if detector not yet registered.

devres (`devres.c`): `devm_extcon_register_notifier` records `(edev, id, nb)` for auto-unregister at consumer device drop.

## Hardening

- **Per-edev mutex** — atomicity around bit-set + notifier broadcast + uevent emission prevents inconsistent observer state.
- **Mutual-exclusion enforced by framework** — defense against detectors leaving contradictory bits set (USB + USB_HOST simultaneously).
- **`supported_cable` validated** — EXTCON_NONE termination required; OOB cable ID rejected at register.
- **`max_supported` bounded** — defense against pathological detectors registering huge cable lists.
- **No spurious uevents** — `_set_state_sync` short-circuits if no bit changed.
- **Per-cable property capability checked** — `_get/_set_property` returns -EPERM if cable did not declare the property.
- **devm-managed subscription** — defense against consumer-leak across rmmod/rebind.
- **Class device refcount** — `extcon_dev_unregister` waits for sysfs grace; concurrent open of /sys/class/extcon/extconN/state safe across teardown.
- **`extcon_dev_list` walk under mutex** — defense against concurrent register/unregister races.
- **Notifier chain blocking-variant** — consumers may sleep in callback; detector never holds atomic ctx across broadcast.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `extcon_dev`, per-cable `extcon_cable`, `extcon_specific_cable_nb`, and per-property value tables; reject userspace copy of per-cable state outside fixed-size sysfs helpers.
- **PAX_KERNEXEC** — extcon core in W^X kernel text; per-cable `device_attribute` show/store ops marked `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `extcon_set_state_sync`, `_set_property_sync`, notifier-broadcast, and `extcon_dev_register` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `extcon_dev` class device, per-cable kobject, and consumer notifier-block ownership; defense against teardown-vs-broadcast races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `extcon_dev`, per-cable `extcon_cable` arrays, per-property value arrays, and uevent envp buffers so a prior cable's property cannot bleed.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs entry; reject user-pointer deref outside canonical `device_attribute` helpers.
- **PAX_RAP / kCFI** — per-cable `device_attribute` show/store, notifier-block `.notifier_call`, and `extcon_dev->dev.release` dispatched via kCFI-typed indirect calls; vtables `__ro_after_init`.
- **GRKERNSEC_HIDESYM** — gate disclosure of per-edev internal pointers (private detector ctx, notifier-head address) in debugfs / tracepoints behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict per-cable state-change banners (commonly "USB cable attached / detached") to CAP_SYSLOG; otherwise unprivileged users can fingerprint device usage.
- **Cable-state PAX_USERCOPY** — per-cable `state` sysfs attribute readers go through `sysfs_emit` (bounded) — no raw `sprintf` on user buffers.
- **Notifier chain RAP** — per-cable `blocking_notifier_head` callbacks dispatched via kCFI-typed indirect; refuse callback registration if `nb->notifier_call` not in any module's text.
- **Sysfs RO for non-root** — `cable.N/{name,state}` is RO; would-be writable test/inject attributes (none today) require CAP_SYS_ADMIN.
- **Uevent allowlist** — uevent envp built from fixed format strings (`CABLE_NAME=...&STATE=...`); refuse cable names containing shell metacharacters; refuse env injection.
- **`max_supported` bounded** — refuse detectors that declare more than EXTCON_NUM cables; prevents heap-spray via supported_cable.
- **Per-property capability strict** — `_set_property` without prior `_set_property_capability(prop)` rejected; defense against capability-confusion.

Rationale: extcon broadcasts to userspace (`udev`) and to in-kernel consumers (charger manager, USB role switch, audio codec). A spoofed cable event can flip USB role from device to host (re-enumerating peripherals on a victim), or alter charger current beyond cable rating (battery damage), or assert HDMI HPD (display side-channels). RAP/kCFI on notifier callbacks + show/store helpers, refcount saturation on the class device, uevent allowlist filtering, and PAX_USERCOPY on sysfs reads turn extcon's broadcast surface into a structurally fenced producer-consumer queue rather than a freeform interrupt vector.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-chip detector drivers (`extcon-max77693.c`, `extcon-usbc-tusb320.c`, etc.) — covered by future Tier-3 if depth warranted; baseline grsec inherits.
- Type-C policy state machine (covered in `drivers/usb/typec/` Tier-3 docs).
- USB role-switch core (covered in `drivers/usb/roles/` Tier-3 docs).
- 32-bit-only paths.
- Implementation code.
