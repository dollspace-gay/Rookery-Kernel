# Tier-3: drivers/usb/core/driver.c — USB driver framework (registration, matching, probe, PM, autosuspend)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/usb/00-overview.md
upstream-paths:
  - drivers/usb/core/driver.c (~2081 lines)
  - include/linux/usb.h (struct usb_driver, struct usb_device_driver, struct usb_device_id)
  - include/linux/mod_devicetable.h (struct usb_device_id, USB_DEVICE_ID_MATCH_*)
  - include/linux/usb/hcd.h (usb_bus_type)
  - include/linux/usb/quirks.h (USB_QUIRK_RESET_RESUME, ...)
-->

## Summary

`drivers/usb/core/driver.c` is the **USB driver framework** layer of usbcore — the glue between the **Linux Driver Model (LDM)** `bus_type` callbacks and the two kinds of USB driver: per-**interface** drivers (`struct usb_driver`, the common case — usb-storage, usbhid, cdc-acm, …) and per-**device** drivers (`struct usb_device_driver`, used for whole-device drivers — `usb_generic_driver` itself, `usbip-host`, debug devices, …).

Per-`usb_register_driver` registers an interface driver: it fills in `new_driver->driver.bus = &usb_bus_type`, sets `.probe = usb_probe_interface`, `.remove = usb_unbind_interface`, `.shutdown = usb_shutdown_interface`, then calls `driver_register()` and creates the per-driver sysfs `new_id` + `remove_id` attribute files (the "dynamic ID" mechanism that lets userspace pin extra `(vendor,product[,class])` triples onto any registered driver via `echo VID PID > /sys/bus/usb/drivers/<name>/new_id`).

Per-`usb_register_device_driver` does the analogous thing for whole-device drivers, with `.probe = usb_probe_device`, `.remove = usb_unbind_device`, and additionally re-scans the bus (`bus_for_each_dev(__usb_bus_reprobe_drivers)`) so a newly-registered specialized device driver can steal a device currently bound to `usb_generic_driver`.

Per-`usb_device_match` is the `bus_type.match` callback: it splits on `is_usb_device(dev)` vs `is_usb_interface(dev)`. For interfaces, it walks the driver's `id_table` via `usb_match_id`, falling back to the **dynamic-IDs list** (`drv->dynids.list`) via `usb_match_dynamic_id`. For whole-device drivers, it consults `usb_driver_applicable` which combines `id_table` (`usb_device_match_id`) AND the driver's optional `udrv->match()` hook.

Per-`usb_match_one_id` is the core matcher: for each `struct usb_device_id`, all bits set in `id->match_flags` must match — vendor (`USB_DEVICE_ID_MATCH_VENDOR`), product (`_PRODUCT`), bcdDevice low/high range (`_DEV_LO`/`_DEV_HI`), device class/subclass/protocol (`_DEV_CLASS`/`_DEV_SUBCLASS`/`_DEV_PROTOCOL`) at the device-descriptor level, then interface class/subclass/protocol/number (`_INT_CLASS`/`_INT_SUBCLASS`/`_INT_PROTOCOL`/`_INT_NUMBER`) at the chosen alt-setting. Per the deliberate quirk: if `bDeviceClass == USB_CLASS_VENDOR_SPEC` and the id record has no vendor match-flag, all interface-level match flags are skipped (vendor-specific class meanings are vendor-private).

Per-`usb_probe_interface` is the LDM `.probe` shim for interface drivers. It refuses to probe if the device is currently `usb_device_is_owned` (claimed exclusively, e.g. by usbip), if `udev->authorized == 0` (per USB-authorization policy — defense against random plug-in attacks), or if `intf->authorized == 0`. It then resolves the matching `struct usb_device_id` via `usb_match_dynamic_id` first then `usb_match_id`, autoresumes the device, sets up per-interface runtime PM (`pm_runtime_set_active`, `pm_runtime_enable` if `supports_autosuspend`), optionally disables hub-initiated LPM if the driver demands (`disable_hub_initiated_lpm`), carries out the deferred Set-Interface to altsetting 0 (`needs_altsetting0`), calls `driver->probe(intf, id)`, and on success marks `intf->condition = USB_INTERFACE_BOUND`.

Per-`usb_probe_device` is the LDM `.probe` shim for device drivers. Unless the driver opts out (`supports_autosuspend`), it autoresumes; if the driver sets `generic_subclass` it chains to `usb_generic_driver_probe` first, then calls the driver's own `probe()`. Importantly, if `udriver->probe()` returns `-ENODEV` and the driver had either an `id_table` or a `match` function (i.e. was a specialized match, not a wildcard), the device is steered back to `usb_generic_driver` via `udev->use_generic_driver = 1; return -EPROBE_DEFER`.

Per-USB PM is layered: system suspend/resume (`usb_suspend`/`usb_resume`/`usb_resume_complete` driving the `usb_bus_type` `pm_ops`), runtime PM (`usb_runtime_suspend`/`usb_runtime_resume`/`usb_runtime_idle`), and autosuspend (`usb_autosuspend_device`, `usb_autoresume_device`, `usb_autopm_get_interface`/`_put_interface` with `_async` and `_no_resume` variants for atomic contexts). The central pipelines are `usb_suspend_both` (suspend all interfaces in reverse order then the device, with full rollback-on-error) and `usb_resume_both` (resume device then all interfaces). Per-device-driver-specific `reset_resume` is the post-power-loss path for devices where state was lost — drivers without `reset_resume` are marked `needs_binding = 1` and rebound after resume by `usb_unbind_and_rebind_marked_interfaces` (the "forced unbind" mechanic also used to evict drivers that lack any PM hooks via `unbind_no_pm_drivers_interfaces`).

Critical for: every USB driver in the tree binds through this file (the entire `drivers/hid/usbhid/`, `drivers/usb/storage/`, `sound/usb/`, `drivers/media/usb/`, `drivers/net/usb/`, `drivers/usb/serial/`, `drivers/usb/class/`, `drivers/usb/misc/` … all funnel through `usb_register_driver` ⟶ `usb_probe_interface`), the sysfs `new_id`/`remove_id` UAPI (per-Documentation/ABI/testing/sysfs-bus-usb), system+runtime PM correctness across the USB bus, and the bus-rescan invariant that demoted a generic-driver-bound device when a specialized driver loads later.

This Tier-3 covers `drivers/usb/core/driver.c` (~2081 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct usb_driver` | per-interface-driver registration record | `UsbDriver` |
| `struct usb_device_driver` | per-device-driver registration record | `UsbDeviceDriver` |
| `struct usb_device_id` | per-match-entry (id_table row) | `UsbDeviceId` |
| `struct usb_dynids` + `struct usb_dynid` | per-driver dynamic-IDs list | `UsbDynIds` / `UsbDynId` |
| `usb_bus_type` | the `bus_type` instance | `UsbBus::BUS_TYPE` |
| `usb_register_driver()` | per-interface-driver registration | `UsbDriver::register` |
| `usb_deregister()` | per-interface-driver deregistration | `UsbDriver::deregister` |
| `usb_register_device_driver()` | per-device-driver registration | `UsbDeviceDriver::register` |
| `usb_deregister_device_driver()` | per-device-driver deregistration | `UsbDeviceDriver::deregister` |
| `usb_store_new_id()` / `usb_show_dynids()` | per-/sys/bus/usb/drivers/<n>/new_id write/show | `UsbDynIds::store_new` / `show` |
| `usb_create_newid_files()` / `usb_remove_newid_files()` | per-driver sysfs new_id/remove_id attach | `UsbDriver::create_newid_files` / `remove_newid_files` |
| `usb_free_dynids()` | per-deregister dynids drain | `UsbDriver::free_dynids` |
| `usb_match_dynamic_id()` | per-dynids-list lookup | `UsbDriver::match_dynamic_id` |
| `usb_match_one_id()` | per-(intf, id) one-shot match | `UsbMatch::match_one_id` |
| `usb_match_one_id_intf()` | per-interface-fields match | `UsbMatch::match_one_id_intf` |
| `usb_match_device()` | per-device-descriptor-fields match | `UsbMatch::match_device` |
| `usb_match_id()` | per-id_table linear scan | `UsbMatch::match_id` |
| `usb_device_match_id()` | per-device-driver id_table scan | `UsbMatch::device_match_id` |
| `usb_driver_applicable()` | per-(udev,udrv) match (id_table + match()) | `UsbMatch::driver_applicable` |
| `usb_device_match()` | `bus_type.match` callback | `UsbBus::device_match` |
| `usb_uevent()` | `bus_type.uevent` callback (PRODUCT=, TYPE=) | `UsbBus::uevent` |
| `usb_probe_device()` | `.probe` for usb_device_driver | `UsbDeviceDriver::probe_dev` |
| `usb_unbind_device()` | `.remove` for usb_device_driver | `UsbDeviceDriver::unbind_dev` |
| `usb_probe_interface()` | `.probe` for usb_driver | `UsbDriver::probe_intf` |
| `usb_unbind_interface()` | `.remove` for usb_driver | `UsbDriver::unbind_intf` |
| `usb_shutdown_interface()` | `.shutdown` for usb_driver | `UsbDriver::shutdown_intf` |
| `usb_driver_claim_interface()` | per-non-probe claim (e.g. usbfs) | `UsbDriver::claim_interface` |
| `usb_driver_release_interface()` | per-explicit release | `UsbDriver::release_interface` |
| `usb_forced_unbind_intf()` | per-forced-rebind unbind | `UsbDriver::forced_unbind_intf` |
| `unbind_marked_interfaces()` / `rebind_marked_interfaces()` | per-needs_binding sweep | `UsbDriver::unbind_marked` / `rebind_marked` |
| `usb_unbind_and_rebind_marked_interfaces()` | per-device rebind pass | `UsbDriver::unbind_and_rebind_marked` |
| `unbind_no_pm_drivers_interfaces()` | per-suspend evict no-PM drivers | `UsbDriver::unbind_no_pm` |
| `__usb_bus_reprobe_drivers()` | per-bus rescan-for-new-driver | `UsbDriver::bus_reprobe_drivers` |
| `is_usb_device_driver()` | per-`drv->probe == usb_probe_device` test | `UsbDeviceDriver::is_device_driver` |
| `usb_suspend_device()` / `usb_resume_device()` | per-device suspend/resume dispatch | `UsbPm::suspend_device` / `resume_device` |
| `usb_suspend_interface()` / `usb_resume_interface()` | per-interface suspend/resume dispatch | `UsbPm::suspend_intf` / `resume_intf` |
| `usb_suspend_both()` / `usb_resume_both()` | per-device+all-interfaces pipeline | `UsbPm::suspend_both` / `resume_both` |
| `choose_wakeup()` | per-suspend wakeup-policy selection | `UsbPm::choose_wakeup` |
| `usb_suspend()` / `usb_resume()` / `usb_resume_complete()` | per-system PM bus callbacks | `UsbPm::sys_suspend` / `sys_resume` / `sys_resume_complete` |
| `usb_runtime_suspend()` / `usb_runtime_resume()` / `usb_runtime_idle()` | per-runtime PM bus callbacks | `UsbPm::rt_suspend` / `rt_resume` / `rt_idle` |
| `usb_enable_autosuspend()` / `usb_disable_autosuspend()` | per-device autosuspend toggle | `UsbPm::enable_autosuspend` / `disable_autosuspend` |
| `usb_autosuspend_device()` / `usb_autoresume_device()` | per-device autosuspend get/put | `UsbPm::autosuspend_device` / `autoresume_device` |
| `usb_autopm_get_interface()` / `usb_autopm_put_interface()` (+ `_async`, `_no_resume`, `_no_suspend`) | per-interface autopm refcount API | `UsbPm::autopm_get_interface*` / `autopm_put_interface*` |
| `autosuspend_check()` | per-eligibility test (usage_count==0, remote wakeup, …) | `UsbPm::autosuspend_check` |
| `usb_enable_usb2_hardware_lpm()` / `usb_disable_usb2_hardware_lpm()` | per-USB-2 HW LPM toggle (HCD callback) | `UsbPm::usb2_hw_lpm_enable` / `disable` |
| `usb_set_usb2_hardware_lpm()` | per-HCD set_usb2_hw_lpm dispatch | `UsbPm::set_usb2_hw_lpm` |

## Compatibility contract

REQ-1: struct usb_driver:
- name: per-driver string (used for /sys/bus/usb/drivers/<name>).
- id_table: per-array of `struct usb_device_id`, terminated by an all-zero entry (modulo nonzero driver_info).
- probe(intf, id): per-interface entry.
- disconnect(intf): per-interface tear-down.
- unlocked_ioctl(intf, code, buf): per-driver ioctl (proxied by usbfs).
- suspend(intf, msg) / resume(intf) / reset_resume(intf): per-PM callbacks.
- pre_reset(intf) / post_reset(intf): per-reset bracketing.
- dynids: per-driver dynamic-IDs list (head + lock).
- soft_unbind: per-disable URB-cancel-on-unbind if device-present.
- disable_hub_initiated_lpm: per-block hub-initiated LPM.
- supports_autosuspend: per-opt-in to runtime PM.
- driver: embedded `struct device_driver` (populated by `usb_register_driver`).

REQ-2: struct usb_device_driver:
- name: per-driver string.
- probe(udev): per-device entry.
- disconnect(udev): per-device tear-down.
- match(udev): per-driver optional predicate (in addition to / instead of id_table).
- suspend(udev, msg) / resume(udev, msg): per-device PM.
- choose_configuration(udev): per-config-selection hook.
- id_table: per-`struct usb_device_id` array (vendor/product/class only at device level).
- generic_subclass: per-cooperate-with-usb_generic_driver flag.
- supports_autosuspend: per-opt-in to runtime PM.
- dev_groups: per-attribute-groups for the device.
- driver: embedded `struct device_driver`.

REQ-3: struct usb_device_id:
- match_flags: per-bitmask of USB_DEVICE_ID_MATCH_* bits (which fields are active).
- idVendor / idProduct: per-vendor/product match.
- bcdDevice_lo / bcdDevice_hi: per-bcdDevice range match (DEV_LO inclusive ≤, DEV_HI inclusive ≥).
- bDeviceClass / bDeviceSubClass / bDeviceProtocol: per-device-descriptor class triple.
- bInterfaceClass / bInterfaceSubClass / bInterfaceProtocol / bInterfaceNumber: per-alt-setting interface tuple.
- driver_info: per-opaque kernel_ulong_t — survives matching, available in probe().

REQ-4: usb_register_driver(new_driver, owner, mod_name):
- /* Forbid registration if USB disabled */
- if usb_disabled(): return -ENODEV.
- /* Fill in LDM hooks */
- new_driver.driver.name = new_driver.name.
- new_driver.driver.bus = &usb_bus_type.
- new_driver.driver.probe = usb_probe_interface.
- new_driver.driver.remove = usb_unbind_interface.
- new_driver.driver.shutdown = usb_shutdown_interface.
- new_driver.driver.owner = owner.
- new_driver.driver.mod_name = mod_name.
- new_driver.driver.dev_groups = new_driver.dev_groups.
- /* Init dynids list */
- INIT_LIST_HEAD(&new_driver.dynids.list).
- /* Register with LDM (triggers probe-rescan) */
- retval = driver_register(&new_driver.driver).
- if retval: return retval.
- /* Create per-driver sysfs new_id + remove_id attrs */
- retval = usb_create_newid_files(new_driver).
- if retval: driver_unregister; return retval.
- return 0.

REQ-5: usb_deregister(driver):
- usb_remove_newid_files(driver).
- driver_unregister(&driver.driver).
- usb_free_dynids(driver) — drain dynids list, free each.

REQ-6: usb_register_device_driver(new_udriver, owner):
- if usb_disabled(): return -ENODEV.
- new_udriver.driver.{name, bus, probe, remove, owner, dev_groups} = {udriver.name, &usb_bus_type, usb_probe_device, usb_unbind_device, owner, udriver.dev_groups}.
- retval = driver_register(&new_udriver.driver).
- if retval==0:
  - /* Rescan: bind any device currently on usb_generic_driver that this new driver wants */
  - bus_for_each_dev(&usb_bus_type, NULL, new_udriver, __usb_bus_reprobe_drivers).
- return retval.

REQ-7: __usb_bus_reprobe_drivers(dev, new_udriver):
- if dev.driver != &usb_generic_driver.driver: return 0.
- udev = to_usb_device(dev).
- if !usb_driver_applicable(udev, new_udriver): return 0.
- device_reprobe(dev) — rebind to new specialized driver.

REQ-8: usb_match_device(dev, id):
- if (MATCH_VENDOR ∧ id.idVendor != dev.descriptor.idVendor): return 0.
- if (MATCH_PRODUCT ∧ id.idProduct != dev.descriptor.idProduct): return 0.
- if (MATCH_DEV_LO ∧ id.bcdDevice_lo > dev.descriptor.bcdDevice): return 0.
- if (MATCH_DEV_HI ∧ id.bcdDevice_hi < dev.descriptor.bcdDevice): return 0.
- if (MATCH_DEV_CLASS ∧ id.bDeviceClass != dev.descriptor.bDeviceClass): return 0.
- if (MATCH_DEV_SUBCLASS ∧ id.bDeviceSubClass != dev.descriptor.bDeviceSubClass): return 0.
- if (MATCH_DEV_PROTOCOL ∧ id.bDeviceProtocol != dev.descriptor.bDeviceProtocol): return 0.
- return 1.

REQ-9: usb_match_one_id_intf(dev, intf, id):
- /* Per-USB-spec quirk: vendor-specific class — skip interface fields unless id specifies vendor */
- if (dev.bDeviceClass == USB_CLASS_VENDOR_SPEC) ∧ !(id.match_flags & MATCH_VENDOR) ∧ (id.match_flags & (MATCH_INT_CLASS | MATCH_INT_SUBCLASS | MATCH_INT_PROTOCOL | MATCH_INT_NUMBER)): return 0.
- if (MATCH_INT_CLASS ∧ id.bInterfaceClass != intf.desc.bInterfaceClass): return 0.
- if (MATCH_INT_SUBCLASS ∧ id.bInterfaceSubClass != intf.desc.bInterfaceSubClass): return 0.
- if (MATCH_INT_PROTOCOL ∧ id.bInterfaceProtocol != intf.desc.bInterfaceProtocol): return 0.
- if (MATCH_INT_NUMBER ∧ id.bInterfaceNumber != intf.desc.bInterfaceNumber): return 0.
- return 1.

REQ-10: usb_match_one_id(interface, id):
- if id == NULL: return 0.
- intf = interface.cur_altsetting.
- dev = interface_to_usbdev(interface).
- if !usb_match_device(dev, id): return 0.
- return usb_match_one_id_intf(dev, intf, id).

REQ-11: usb_match_id(interface, id_table):
- if id_table == NULL: return NULL.
- /* Walk until terminator (all match keys + driver_info zero) */
- for id from id_table while (id.idVendor ∨ id.idProduct ∨ id.bDeviceClass ∨ id.bInterfaceClass ∨ id.driver_info):
  - if usb_match_one_id(interface, id): return id.
- return NULL.

REQ-12: usb_device_match_id(udev, id_table):
- if !id_table: return NULL.
- for id while (id.idVendor ∨ id.idProduct):
  - if usb_match_device(udev, id): return id.
- return NULL.

REQ-13: usb_driver_applicable(udev, udrv):
- /* Both id_table AND match() — both must say yes */
- if udrv.id_table ∧ udrv.match: return usb_device_match_id(udev, udrv.id_table) != NULL ∧ udrv.match(udev).
- if udrv.id_table: return usb_device_match_id(udev, udrv.id_table) != NULL.
- if udrv.match: return udrv.match(udev).
- return false.

REQ-14: usb_match_dynamic_id(intf, drv):
- guard(mutex)(&usb_dynids_lock).
- for_each_entry(dynid, &drv.dynids.list):
  - if usb_match_one_id(intf, &dynid.id): return &dynid.id.
- return NULL.

REQ-15: usb_device_match(dev, drv) (bus_type.match):
- /* USB device vs USB interface split */
- if is_usb_device(dev):
  - if !is_usb_device_driver(drv): return 0.
  - udrv = to_usb_device_driver(drv).
  - /* Wildcard match: no id_table AND no match() — defer to .probe */
  - if !udrv.id_table ∧ !udrv.match: return 1.
  - return usb_driver_applicable(to_usb_device(dev), udrv).
- if is_usb_interface(dev):
  - if is_usb_device_driver(drv): return 0.
  - usb_drv = to_usb_driver(drv).
  - if usb_match_id(intf, usb_drv.id_table): return 1.
  - if usb_match_dynamic_id(intf, usb_drv): return 1.
- return 0.

REQ-16: usb_store_new_id(dynids, id_table, driver, buf, count):
- /* Userspace echoes "VID PID [Class [refVID refPID]]" into /sys/bus/usb/drivers/<n>/new_id */
- sscanf "%x %x %x %x %x" → idVendor, idProduct, bInterfaceClass, refVendor, refProduct.
- if fields < 2: return -EINVAL.
- Allocate `struct usb_dynid`.
- dynid.id.{idVendor, idProduct, match_flags = USB_DEVICE_ID_MATCH_DEVICE}.
- if fields > 2 ∧ bInterfaceClass:
  - if bInterfaceClass > 255: -EINVAL.
  - dynid.id.bInterfaceClass = (u8) bInterfaceClass.
  - dynid.id.match_flags |= USB_DEVICE_ID_MATCH_INT_CLASS.
- if fields > 4:
  - /* Lookup (refVID, refPID) in id_table to copy driver_info */
  - walk id_table; if !found: -ENODEV.
  - dynid.id.driver_info = found_id.driver_info.
- mutex_lock(&usb_dynids_lock); list_add_tail(&dynid.node, &dynids.list); mutex_unlock.
- /* Re-attach: re-scan bus for matches */
- driver_attach(driver).
- return count.

REQ-17: usb_probe_interface(dev) (bus_type.probe for interfaces):
- driver = to_usb_driver(dev.driver).
- intf = to_usb_interface(dev).
- udev = interface_to_usbdev(intf).
- intf.needs_binding = 0.
- /* Owner-policy gate: usbfs/usbip exclusive ownership */
- if usb_device_is_owned(udev): return -ENODEV.
- /* Authorization gate: per /sys/bus/usb/devices/<n>/authorized */
- if udev.authorized == 0: return -ENODEV.
- if intf.authorized == 0: return -ENODEV.
- /* Match → id */
- id = usb_match_dynamic_id(intf, driver) ∨ usb_match_id(intf, driver.id_table).
- if !id: return -ENODEV.
- /* PM: pin device awake for probe */
- usb_autoresume_device(udev).
- intf.condition = USB_INTERFACE_BINDING.
- /* Per-interface runtime PM */
- pm_runtime_set_active(dev).
- pm_suspend_ignore_children(dev, false).
- if driver.supports_autosuspend: pm_runtime_enable(dev).
- /* Optional: disable hub-initiated LPM */
- if driver.disable_hub_initiated_lpm: usb_unlocked_disable_lpm(udev).
- /* Deferred altsetting-0 install */
- if intf.needs_altsetting0: usb_set_interface(udev, intf.altsetting[0].desc.bInterfaceNumber, 0).
- /* Driver probe */
- error = driver.probe(intf, id).
- if error: goto err.
- intf.condition = USB_INTERFACE_BOUND.
- /* Re-enable LPM if previously disabled */
- if lpm_disable_succeeded: usb_unlocked_enable_lpm(udev).
- usb_autosuspend_device(udev).
- return 0.
- err: usb_set_intfdata(intf, NULL); intf.needs_remote_wakeup = 0; intf.condition = USB_INTERFACE_UNBOUND; restore LPM; disable+suspend runtime PM; usb_autosuspend_device.

REQ-18: usb_probe_device(dev) (bus_type.probe for whole devices):
- udriver = to_usb_device_driver(dev.driver).
- udev = to_usb_device(dev).
- /* PM */
- if !udriver.supports_autosuspend: usb_autoresume_device(udev).
- /* Chain to generic driver first if requested */
- if udriver.generic_subclass: usb_generic_driver_probe(udev).
- /* Driver probe */
- if udriver.probe: error = udriver.probe(udev).
- else if !udriver.generic_subclass: error = -EINVAL.
- /* Demotion-to-generic policy */
- if error == -ENODEV ∧ udriver != &usb_generic_driver ∧ (udriver.id_table ∨ udriver.match):
  - udev.use_generic_driver = 1.
  - return -EPROBE_DEFER.
- return error.

REQ-19: usb_unbind_interface(dev):
- driver = to_usb_driver(dev.driver).
- intf.condition = USB_INTERFACE_UNBINDING.
- /* Optional: disable hub LPM */
- if driver.disable_hub_initiated_lpm: usb_unlocked_disable_lpm(udev).
- /* Cancel URBs unless soft-unbind+still-attached */
- if !driver.soft_unbind ∨ udev.state == USB_STATE_NOTATTACHED: usb_disable_interface(udev, intf, false).
- driver.disconnect(intf).
- /* Free streams */
- usb_free_streams(intf, …) if any.
- /* Restore altsetting 0 if we can; otherwise defer via needs_altsetting0 */
- if cur_alt.bAlternateSetting == 0: usb_enable_interface(udev, intf, false).
- else if can-set: usb_set_interface(udev, alt0, 0).
- else: intf.needs_altsetting0 = 1.
- usb_set_intfdata(intf, NULL).
- intf.condition = USB_INTERFACE_UNBOUND; intf.needs_remote_wakeup = 0.
- Restore LPM; disable+suspend runtime PM; usb_autosuspend_device.

REQ-20: usb_driver_claim_interface(driver, iface, data):
- /* Bypass match — used by usbfs (any user-claimed interface) */
- if !iface: -ENODEV.
- if iface.dev.driver: -EBUSY (already bound).
- if !iface.authorized: -ENODEV.
- iface.dev.driver = &driver.driver.
- usb_set_intfdata(iface, data).
- iface.condition = USB_INTERFACE_BOUND.
- /* Runtime PM */
- pm_suspend_ignore_children(dev, false).
- if driver.supports_autosuspend: pm_runtime_enable(dev) else pm_runtime_set_active(dev).
- if device_is_registered(dev): retval = device_bind_driver(dev).
- on retval ≠ 0: revert all.

REQ-21: usb_driver_release_interface(driver, iface):
- if iface.dev.driver != &driver.driver: return (not ours).
- if iface.condition != USB_INTERFACE_BOUND: return (e.g. in disconnect).
- iface.condition = USB_INTERFACE_UNBINDING.
- if device_is_registered(dev): device_release_driver(dev) (triggers usb_unbind_interface).
- else: device_lock; usb_unbind_interface; dev.driver = NULL; device_unlock.

REQ-22: usb_uevent(dev, env) (bus_type.uevent):
- if usb_dev.devnum < 0: -ENODEV.
- if !usb_dev.bus: -ENODEV.
- add "PRODUCT=<vid>/<pid>/<bcd>".
- add "TYPE=<class>/<subclass>/<proto>".

REQ-23: usb_suspend_both(udev, msg) (system+autosuspend pipeline):
- if state ∈ {NOTATTACHED, SUSPENDED}: done.
- /* Offload check (e.g. USB-audio in hardware) */
- usb_offload_set_pm_locked(udev, true).
- if msg.event == PM_EVENT_SUSPEND ∧ usb_offload_check(udev): offload_active = true.
- /* Suspend interfaces in reverse order */
- for i = nInterfaces-1; i ≥ 0; --i:
  - intf = actconfig.interface[i].
  - if offload_active ∧ intf.needs_remote_wakeup: skip.
  - status = usb_suspend_interface(udev, intf, msg).
  - if !PMSG_IS_AUTO(msg): status = 0 (ignore in sys-sleep).
  - if status != 0: break.
- /* Suspend device */
- if status == 0 ∧ !offload_active: status = usb_suspend_device(udev, msg).
- /* Non-root-hub during sys-sleep: ignore errors */
- if udev.parent ∧ !PMSG_IS_AUTO(msg): status = 0.
- /* Health probe on persistent error */
- if status ∧ status != -EBUSY: usb_get_std_status(udev, ...) sanity.
- /* Rollback: resume already-suspended interfaces */
- if status != 0: while (++i < n) usb_resume_interface(udev, actconfig.interface[i], reversed_msg, 0).
- /* Success: pin can_submit = 0 + flush */
- else: udev.can_submit = 0; for ep_in/ep_out: usb_hcd_flush_endpoint.

REQ-24: usb_resume_both(udev, msg):
- if state == NOTATTACHED: -ENODEV.
- udev.can_submit = 1.
- /* Resume device */
- if state == SUSPENDED ∨ udev.reset_resume: status = usb_resume_device(udev, msg).
- /* Resume interfaces (forward order) */
- if status == 0 ∧ udev.actconfig:
  - for i = 0 .. nInterfaces-1:
    - intf = actconfig.interface[i].
    - if offload_active ∧ intf.needs_remote_wakeup: skip.
    - usb_resume_interface(udev, intf, msg, udev.reset_resume).
- usb_mark_last_busy(udev).
- if !status: udev.reset_resume = 0.

REQ-25: usb_resume_interface(udev, intf, msg, reset_resume):
- if state == NOTATTACHED ∨ intf.condition == UNBINDING ∨ intf.needs_binding: skip.
- if intf.condition == UNBOUND: deferred altsetting-0 install if needed; skip.
- driver = to_usb_driver(intf.dev.driver).
- /* reset_resume path */
- if reset_resume:
  - if driver.reset_resume: driver.reset_resume(intf).
  - else: intf.needs_binding = 1 (will be unbound + rebound).
- else: driver.resume(intf).

REQ-26: choose_wakeup(udev, msg):
- if msg.event ∈ {FREEZE, QUIESCE}: w = 0.
- else: w = device_may_wakeup(&udev.dev).
- /* Autoresume if wakeup-setting mismatches current suspended state */
- if state == SUSPENDED ∧ w != udev.do_remote_wakeup: pm_runtime_resume(&udev.dev).
- udev.do_remote_wakeup = w.

REQ-27: usb_suspend(dev, msg) (system-PM bus callback):
- udev = to_usb_device(dev).
- /* Evict any interface driver lacking PM callbacks first */
- unbind_no_pm_drivers_interfaces(udev).
- choose_wakeup(udev, msg).
- r = usb_suspend_both(udev, msg).

REQ-28: usb_resume(dev, msg) / usb_resume_complete(dev):
- Symmetric. After resume, `usb_resume_complete` drives the `usb_unbind_and_rebind_marked_interfaces` pass for drivers that lacked reset_resume.

REQ-29: usb_runtime_suspend(dev) / _resume(dev) / _idle(dev):
- _suspend: if autosuspend_check != 0: -EAGAIN; else usb_suspend_both(PMSG_AUTO_SUSPEND); coerce errors to -EBUSY for non-root-hubs.
- _resume: usb_resume_both(PMSG_AUTO_RESUME).
- _idle: if autosuspend_check == 0: pm_runtime_autosuspend; return -EBUSY (tell PM core not to suspend now).

REQ-30: autosuspend_check(udev):
- if state == NOTATTACHED: -ENODEV.
- For each interface:
  - if power.disable_depth: skip (unbound or no-autosuspend).
  - if power.usage_count > 0: -EBUSY.
  - w |= intf.needs_remote_wakeup.
  - if udev.quirks & USB_QUIRK_RESET_RESUME:
    - driver = to_usb_driver(intf.dev.driver).
    - if !driver.reset_resume ∨ intf.needs_remote_wakeup: -EOPNOTSUPP.
- if w ∧ !device_can_wakeup(&udev.dev): -EOPNOTSUPP.
- if w ∧ udev.parent == root_hub ∧ bus_to_hcd.cant_recv_wakeups: -EOPNOTSUPP.
- udev.do_remote_wakeup = w; return 0.

REQ-31: usb_autosuspend_device(udev): pm_runtime_put_sync_autosuspend(&udev.dev).
REQ-32: usb_autoresume_device(udev): pm_runtime_resume_and_get(&udev.dev).
REQ-33: usb_autopm_put_interface(intf) (+ _async, _no_suspend): pm_runtime_put_sync / put / put_noidle on intf.dev.
REQ-34: usb_autopm_get_interface(intf) (+ _async, _no_resume): pm_runtime_resume_and_get / get / get_noresume on intf.dev.

REQ-35: usb_bus_type:
- name = "usb".
- match = usb_device_match.
- uevent = usb_uevent.
- need_parent_lock = true.

## Acceptance Criteria

- [ ] AC-1: `usb_register_driver` installs `.probe = usb_probe_interface`, `.remove = usb_unbind_interface`, `.shutdown = usb_shutdown_interface` on `new_driver->driver`, registers it with the LDM, creates per-driver `new_id`+`remove_id` sysfs attrs.
- [ ] AC-2: `usb_register_device_driver` installs `.probe = usb_probe_device` and triggers `bus_for_each_dev(__usb_bus_reprobe_drivers)` so generic-driver-bound devices are re-probed.
- [ ] AC-3: `usb_match_one_id` against an id with `MATCH_VENDOR|MATCH_PRODUCT` returns 1 iff both descriptor fields match.
- [ ] AC-4: `usb_match_one_id_intf` skips all `MATCH_INT_*` checks if `bDeviceClass == USB_CLASS_VENDOR_SPEC` and the id has no `MATCH_VENDOR` flag.
- [ ] AC-5: `usb_match_id` walks `id_table` until the all-zero (modulo nonzero `driver_info`) terminator; returns NULL on no match.
- [ ] AC-6: `usb_store_new_id` accepts "VID PID", "VID PID Class", and "VID PID Class refVID refPID" forms; rejects fields<2 with -EINVAL; rejects Class>255 with -EINVAL; rejects refVID/refPID not found in id_table with -ENODEV.
- [ ] AC-7: After echoing into `/sys/bus/usb/drivers/<name>/new_id`, `driver_attach` is invoked to re-scan and bind matching unbound devices.
- [ ] AC-8: `usb_probe_interface` rejects with -ENODEV if `udev->authorized == 0` or `intf->authorized == 0` or the device is owned.
- [ ] AC-9: `usb_probe_interface` autoresumes the device, probes the driver, autosuspends on success or full-rollback on error.
- [ ] AC-10: `usb_probe_device` returns -EPROBE_DEFER (with `use_generic_driver = 1`) when a specialized driver's probe returns -ENODEV.
- [ ] AC-11: `usb_unbind_interface` cancels in-flight URBs unless `soft_unbind ∧ udev->state != NOTATTACHED`.
- [ ] AC-12: `usb_suspend_both` suspends interfaces reverse-order then device, resumes already-suspended interfaces on failure (autosuspend) or ignores errors in system sleep.
- [ ] AC-13: `usb_resume_both` resumes device then interfaces forward-order; `reset_resume` path is taken if `udev->reset_resume` is set.
- [ ] AC-14: Drivers without `reset_resume` get `intf->needs_binding = 1`, triggering `usb_unbind_and_rebind_marked_interfaces` in `usb_resume_complete`.
- [ ] AC-15: `autosuspend_check` returns -EBUSY if any interface's `power.usage_count > 0`; returns -EOPNOTSUPP if a remote-wakeup-needing interface lacks bus wakeup capability.
- [ ] AC-16: `usb_autopm_get_interface` increments `intf->dev.power.usage_count` and synchronously autoresumes; `_async` variant queues a resume and may be called in atomic context; `_no_resume` increments without resuming.
- [ ] AC-17: `usb_driver_claim_interface` (called by usbfs) binds without going through `usb_probe_interface`; refuses if interface already has a driver (-EBUSY) or is unauthorized (-ENODEV).

## Architecture

```
struct UsbDriver {
  name: &'static str,
  id_table: &'static [UsbDeviceId],
  dynids: SpinMutex<DynIdsList>,
  probe: fn(&mut UsbInterface, &UsbDeviceId) -> Result<(), Errno>,
  disconnect: fn(&mut UsbInterface),
  unlocked_ioctl: Option<fn(&mut UsbInterface, u32, *mut u8) -> Result<i32, Errno>>,
  suspend: Option<fn(&mut UsbInterface, PmMessage) -> Result<(), Errno>>,
  resume: Option<fn(&mut UsbInterface) -> Result<(), Errno>>,
  reset_resume: Option<fn(&mut UsbInterface) -> Result<(), Errno>>,
  pre_reset: Option<fn(&mut UsbInterface) -> Result<(), Errno>>,
  post_reset: Option<fn(&mut UsbInterface) -> Result<(), Errno>>,
  supports_autosuspend: bool,
  disable_hub_initiated_lpm: bool,
  soft_unbind: bool,
  dev_groups: &'static [AttributeGroup],
  driver: DeviceDriver,           // LDM embed
}

struct UsbDeviceDriver {
  name: &'static str,
  id_table: Option<&'static [UsbDeviceId]>,
  probe: Option<fn(&mut UsbDevice) -> Result<(), Errno>>,
  disconnect: Option<fn(&mut UsbDevice)>,
  match: Option<fn(&UsbDevice) -> bool>,
  suspend: Option<fn(&mut UsbDevice, PmMessage) -> Result<(), Errno>>,
  resume: Option<fn(&mut UsbDevice, PmMessage) -> Result<(), Errno>>,
  choose_configuration: Option<fn(&UsbDevice) -> i32>,
  generic_subclass: bool,
  supports_autosuspend: bool,
  dev_groups: &'static [AttributeGroup],
  driver: DeviceDriver,
}

struct UsbDeviceId {
  match_flags: u16,                // USB_DEVICE_ID_MATCH_*
  idVendor: u16,
  idProduct: u16,
  bcdDevice_lo: u16,
  bcdDevice_hi: u16,
  bDeviceClass: u8,
  bDeviceSubClass: u8,
  bDeviceProtocol: u8,
  bInterfaceClass: u8,
  bInterfaceSubClass: u8,
  bInterfaceProtocol: u8,
  bInterfaceNumber: u8,
  driver_info: usize,              // kernel_ulong_t
}

struct DynIdsList {
  list: ListHead<UsbDynId>,
}

struct UsbDynId {
  node: ListLink,
  id: UsbDeviceId,
}
```

`UsbDriver::register(self, owner, mod_name) -> Result<(), Errno>`:
1. if usb_disabled(): return Err(ENODEV).
2. self.driver.{name = self.name, bus = &UsbBus::BUS_TYPE, probe = UsbDriver::probe_intf, remove = UsbDriver::unbind_intf, shutdown = UsbDriver::shutdown_intf, owner, mod_name, dev_groups = self.dev_groups}.
3. self.dynids = SpinMutex::new(DynIdsList::new()).
4. driver_register(&self.driver)?
5. UsbDriver::create_newid_files(self)?  /* sysfs new_id + remove_id attrs */
6. Ok(())

`UsbDriver::probe_intf(dev) -> Result<(), Errno>`:
1. driver = to_usb_driver(dev.driver).
2. intf = to_usb_interface(dev).
3. udev = interface_to_usbdev(intf).
4. intf.needs_binding = false.
5. if usb_device_is_owned(udev): return Err(ENODEV).
6. if udev.authorized == 0: return Err(ENODEV).
7. if intf.authorized == 0: return Err(ENODEV).
8. id = UsbDriver::match_dynamic_id(intf, driver).or_else(|| UsbMatch::match_id(intf, driver.id_table)).ok_or(ENODEV)?
9. UsbPm::autoresume_device(udev)?
10. intf.condition = UsbInterfaceCondition::Binding.
11. pm_runtime_set_active(dev); pm_suspend_ignore_children(dev, false).
12. if driver.supports_autosuspend: pm_runtime_enable(dev).
13. if driver.disable_hub_initiated_lpm: lpm_disable_error = usb_unlocked_disable_lpm(udev).
14. if intf.needs_altsetting0: usb_set_interface(udev, intf.altsetting[0].desc.bInterfaceNumber, 0)?
15. (driver.probe)(intf, id) → on Err: jump err.
16. intf.condition = UsbInterfaceCondition::Bound.
17. if lpm_disable_error == 0: usb_unlocked_enable_lpm(udev).
18. UsbPm::autosuspend_device(udev).
19. Ok(())
20. err: usb_set_intfdata(intf, ptr::null_mut()); intf.needs_remote_wakeup = false; intf.condition = UsbInterfaceCondition::Unbound; restore LPM; if supports_autosuspend: pm_runtime_disable; pm_runtime_set_suspended; UsbPm::autosuspend_device.

`UsbDeviceDriver::probe_dev(dev) -> Result<(), Errno>`:
1. udriver = to_usb_device_driver(dev.driver).
2. udev = to_usb_device(dev).
3. if !udriver.supports_autosuspend: UsbPm::autoresume_device(udev)?
4. if udriver.generic_subclass: usb_generic_driver_probe(udev)?
5. error = match udriver.probe { Some(p) => p(udev), None if !udriver.generic_subclass => Err(EINVAL), _ => Ok(()) }.
6. if error == Err(ENODEV) ∧ udriver != &usb_generic_driver ∧ (udriver.id_table.is_some() ∨ udriver.match.is_some()):
   - udev.use_generic_driver = true.
   - return Err(EPROBE_DEFER).
7. error.

`UsbMatch::match_one_id(interface, id) -> bool`:
1. if id.is_null(): return false.
2. intf = interface.cur_altsetting.
3. dev = interface_to_usbdev(interface).
4. if !UsbMatch::match_device(dev, id): return false.
5. UsbMatch::match_one_id_intf(dev, intf, id).

`UsbMatch::match_device(dev, id) -> bool`:
1. if id.match_flags & MATCH_VENDOR ∧ id.idVendor != le16(dev.descriptor.idVendor): return false.
2. if id.match_flags & MATCH_PRODUCT ∧ id.idProduct != le16(dev.descriptor.idProduct): return false.
3. if id.match_flags & MATCH_DEV_LO ∧ id.bcdDevice_lo > le16(dev.descriptor.bcdDevice): return false.
4. if id.match_flags & MATCH_DEV_HI ∧ id.bcdDevice_hi < le16(dev.descriptor.bcdDevice): return false.
5. if id.match_flags & MATCH_DEV_CLASS ∧ id.bDeviceClass != dev.descriptor.bDeviceClass: return false.
6. if id.match_flags & MATCH_DEV_SUBCLASS ∧ id.bDeviceSubClass != dev.descriptor.bDeviceSubClass: return false.
7. if id.match_flags & MATCH_DEV_PROTOCOL ∧ id.bDeviceProtocol != dev.descriptor.bDeviceProtocol: return false.
8. true.

`UsbMatch::match_one_id_intf(dev, intf, id) -> bool`:
1. /* Vendor-spec quirk: skip int-level fields unless id has MATCH_VENDOR */
2. if dev.descriptor.bDeviceClass == USB_CLASS_VENDOR_SPEC ∧ !(id.match_flags & MATCH_VENDOR) ∧ (id.match_flags & (MATCH_INT_CLASS | MATCH_INT_SUBCLASS | MATCH_INT_PROTOCOL | MATCH_INT_NUMBER)) != 0:
   - return false.
3. if id.match_flags & MATCH_INT_CLASS ∧ id.bInterfaceClass != intf.desc.bInterfaceClass: return false.
4. if id.match_flags & MATCH_INT_SUBCLASS ∧ id.bInterfaceSubClass != intf.desc.bInterfaceSubClass: return false.
5. if id.match_flags & MATCH_INT_PROTOCOL ∧ id.bInterfaceProtocol != intf.desc.bInterfaceProtocol: return false.
6. if id.match_flags & MATCH_INT_NUMBER ∧ id.bInterfaceNumber != intf.desc.bInterfaceNumber: return false.
7. true.

`UsbBus::device_match(dev, drv) -> bool`:
1. if is_usb_device(dev):
   - if !UsbDeviceDriver::is_device_driver(drv): return false.
   - udev = to_usb_device(dev); udrv = to_usb_device_driver(drv).
   - if udrv.id_table.is_none() ∧ udrv.match.is_none(): return true.   /* wildcard */
   - UsbMatch::driver_applicable(udev, udrv).
2. else if is_usb_interface(dev):
   - if UsbDeviceDriver::is_device_driver(drv): return false.
   - intf = to_usb_interface(dev); usb_drv = to_usb_driver(drv).
   - if UsbMatch::match_id(intf, usb_drv.id_table).is_some(): return true.
   - if UsbDriver::match_dynamic_id(intf, usb_drv).is_some(): return true.
3. false.

`UsbPm::suspend_both(udev, msg) -> Result<(), Errno>`:
1. if udev.state ∈ {NOTATTACHED, SUSPENDED}: return Ok(()).
2. usb_offload_set_pm_locked(udev, true).
3. let offload_active = (msg.event == PM_EVENT_SUSPEND) ∧ usb_offload_check(udev).
4. let n = udev.actconfig.map(|c| c.desc.bNumInterfaces).unwrap_or(0).
5. let mut i = n;
6. while i > 0:
   - i -= 1.
   - intf = udev.actconfig.unwrap().interface[i].
   - if offload_active ∧ intf.needs_remote_wakeup: continue.
   - status = UsbPm::suspend_intf(udev, intf, msg).
   - if !pmsg_is_auto(msg): status = Ok(()).   /* system sleep ignores errors */
   - if status.is_err(): break.
7. if status.is_ok() ∧ !offload_active: status = UsbPm::suspend_device(udev, msg).
8. if udev.parent.is_some() ∧ !pmsg_is_auto(msg): status = Ok(()).
9. if status.is_err() ∧ status != Err(EBUSY): usb_get_std_status(udev, …) sanity check.
10. if status.is_err():
    - /* Rollback: resume interfaces we suspended */
    - msg.event ^= PM_EVENT_SUSPEND | PM_EVENT_RESUME.
    - while ++i < n: UsbPm::resume_intf(udev, actconfig.interface[i], msg, false).
11. else:
    - udev.can_submit = false.
    - if !offload_active: for ep in 0..16: usb_hcd_flush_endpoint(udev, ep_out[ep]); usb_hcd_flush_endpoint(udev, ep_in[ep]).
12. status.

`UsbPm::autosuspend_check(udev) -> Result<(), Errno>`:
1. if udev.state == NOTATTACHED: return Err(ENODEV).
2. let mut w = false.
3. if let Some(cfg) = udev.actconfig:
   - for intf in cfg.interfaces():
     - if intf.dev.power.disable_depth > 0: continue.
     - if intf.dev.power.usage_count > 0: return Err(EBUSY).
     - w |= intf.needs_remote_wakeup.
     - if udev.quirks & USB_QUIRK_RESET_RESUME != 0:
       - driver = to_usb_driver(intf.dev.driver).
       - if driver.reset_resume.is_none() ∨ intf.needs_remote_wakeup: return Err(EOPNOTSUPP).
4. if w ∧ !device_can_wakeup(&udev.dev): return Err(EOPNOTSUPP).
5. if w ∧ ptr::eq(udev.parent, udev.bus.root_hub) ∧ bus_to_hcd(udev.bus).cant_recv_wakeups: return Err(EOPNOTSUPP).
6. udev.do_remote_wakeup = w.
7. Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `register_bus_field_correctly_set` | INVARIANT | per-usb_register_driver: driver.bus == &usb_bus_type. |
| `register_probe_field_is_usb_probe_interface` | INVARIANT | per-usb_register_driver: driver.probe == usb_probe_interface. |
| `device_register_probe_field_is_usb_probe_device` | INVARIANT | per-usb_register_device_driver: driver.probe == usb_probe_device. |
| `match_one_id_null_id_returns_false` | INVARIANT | per-usb_match_one_id: id == NULL ⟹ 0. |
| `vendor_spec_skip_intf_when_no_vendor_flag` | INVARIANT | per-usb_match_one_id_intf: VENDOR_SPEC ∧ !MATCH_VENDOR ∧ any-INT-flag ⟹ 0. |
| `dynids_lock_held_during_match_dynamic_id` | INVARIANT | per-usb_match_dynamic_id: usb_dynids_lock held. |
| `probe_intf_authorization_gate` | INVARIANT | per-usb_probe_interface: udev.authorized==0 ∨ intf.authorized==0 ⟹ -ENODEV (no probe call). |
| `unbind_interface_intfdata_cleared` | INVARIANT | per-usb_unbind_interface: usb_set_intfdata(intf, NULL) on exit. |
| `suspend_both_resume_on_error_balances` | INVARIANT | per-usb_suspend_both: # interfaces resumed in rollback == # suspended before error. |
| `autosuspend_check_usage_count_gate` | INVARIANT | per-autosuspend_check: usage_count > 0 ⟹ -EBUSY. |
| `claim_interface_refuses_already_bound` | INVARIANT | per-usb_driver_claim_interface: iface.dev.driver != NULL ⟹ -EBUSY. |
| `release_interface_only_releases_own` | INVARIANT | per-usb_driver_release_interface: iface.dev.driver != &driver.driver ⟹ no-op. |

### Layer 2: TLA+

`drivers/usb/core/driver.tla`:
- Per-register + per-bus-rescan + per-probe + per-unbind + per-suspend + per-resume + per-autosuspend.
- Properties:
  - `safety_probe_only_after_match` — per-probe_interface: driver.probe called only after usb_match_id ∨ usb_match_dynamic_id returns Some.
  - `safety_probe_only_when_authorized` — per-probe_interface: udev.authorized = 1 ∧ intf.authorized = 1 at probe-call site.
  - `safety_suspend_then_resume_count_balanced` — per-suspend_both: rollback path resumes exactly the interfaces that were suspended.
  - `safety_unbind_clears_intfdata` — per-unbind_interface: usb_set_intfdata(intf, NULL) ⟹ intfdata cleared before condition transition.
  - `safety_dynids_atomic_add` — per-usb_store_new_id: dynid append visible-or-not (no torn state).
  - `safety_generic_driver_demotion` — per-probe_device: -ENODEV from specialized driver ∧ udriver != usb_generic_driver ∧ (id_table ∨ match) ⟹ use_generic_driver = 1.
  - `liveness_register_eventually_attaches_matching_devices` — per-usb_register_driver: any plugged matching device is eventually bound (modulo authorization).
  - `liveness_autosuspend_check_eventually_returns` — per-autosuspend_check: terminates (bounded by nInterfaces).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `UsbDriver::register` post: driver registered ∨ new_id/remove_id files absent | `UsbDriver::register` |
| `UsbDriver::register` post: dynids list initialized empty | `UsbDriver::register` |
| `UsbDeviceDriver::register` post: re-probe pass invoked across the bus | `UsbDeviceDriver::register` |
| `UsbMatch::match_id` post: returned id satisfies all id.match_flags fields | `UsbMatch::match_id` |
| `UsbMatch::match_one_id_intf` post: returns false on vendor-spec quirk | `UsbMatch::match_one_id_intf` |
| `UsbDriver::probe_intf` post: condition transitions Binding→Bound on success, Binding→Unbound on error | `UsbDriver::probe_intf` |
| `UsbDriver::probe_intf` post: PM refcount net-zero (autoresume balanced by autosuspend on both paths) | `UsbDriver::probe_intf` |
| `UsbDeviceDriver::probe_dev` post: -EPROBE_DEFER ⟹ udev.use_generic_driver = 1 | `UsbDeviceDriver::probe_dev` |
| `UsbDriver::unbind_intf` post: condition = Unbound; intfdata = NULL; runtime PM disabled+suspended | `UsbDriver::unbind_intf` |
| `UsbPm::suspend_both` post: success ⟹ udev.can_submit = 0 ∧ all interfaces suspended | `UsbPm::suspend_both` |
| `UsbPm::resume_both` post: success ⟹ udev.reset_resume = 0 | `UsbPm::resume_both` |
| `UsbPm::autosuspend_check` post: usage_count == 0 on every enabled intf | `UsbPm::autosuspend_check` |
| `UsbDriver::claim_interface` post: success ⟹ iface.dev.driver == &driver.driver ∧ condition = Bound | `UsbDriver::claim_interface` |
| `UsbDriver::release_interface` post: success ⟹ iface.dev.driver == NULL ∨ unchanged-if-not-ours | `UsbDriver::release_interface` |

### Layer 4: Verus/Creusot functional

`Per-usb_register_driver → driver_register → usb_create_newid_files → driver-core probe-rescan → for each matching unbound interface: usb_probe_interface → driver.probe; per-usb_register_device_driver → driver_register → __usb_bus_reprobe_drivers → device_reprobe of generic-bound devices; per-usb_match_id walks id_table left-to-right with usb_match_one_id; per-usb_match_dynamic_id walks dynids list; per-usb_suspend_both reverse-suspends interfaces then device with rollback-on-failure (autosuspend) or ignore-errors (system sleep); per-usb_resume_both forward-resumes device then interfaces (with reset_resume path for power-loss); per-autosuspend_check gates runtime suspend on usage_count + remote-wakeup capability + RESET_RESUME quirk; per-usb_autopm_get/put_interface refcount the per-intf runtime PM` semantic equivalence: per-Documentation/driver-api/usb/{usb,power-management}.rst + per-Documentation/ABI/testing/sysfs-bus-usb.

## Hardening

(Inherits row-1 features from `drivers/usb/00-overview.md` § Hardening.)

USB-driver-framework reinforcement:

- **Per-driver-name uniqueness enforced by LDM** — defense against per-collision-shadow attacks.
- **Per-authorization gate (udev.authorized ∧ intf.authorized) at probe-entry** — defense against per-malicious-HID auto-binding (per /sys/bus/usb/devices/<n>/authorized; per `authorized_default` sysfs knob).
- **Per-`usb_device_is_owned` exclusive-claim respected** — defense against per-usbip / usbfs / driver-grab races.
- **Per-id_table terminator strictly enforced** — defense against per-out-of-bounds walk of a missing-sentinel array.
- **Per-USB_CLASS_VENDOR_SPEC interface-fields skip** — defense against per-spurious-match on vendor-private classes (per USB spec).
- **Per-dynids `mutex_lock(&usb_dynids_lock)`** — defense against per-concurrent-new_id list-corruption.
- **Per-`new_id` CAP_SYS_ADMIN write gate (sysfs perms 0200)** — defense against per-unprivileged ID-injection.
- **Per-`use_generic_driver`+`-EPROBE_DEFER` demotion** — defense against per-half-bound-specialized-driver locking out fallback.
- **Per-`soft_unbind` URB-flush policy** — defense against per-stuck-pending-URB on driver-rebind.
- **Per-suspend rollback (resume already-suspended interfaces)** — defense against per-half-suspended-bus inconsistency.
- **Per-`reset_resume` mandatory for RESET_RESUME quirk + autosuspend** — defense against per-state-loss undetected.
- **Per-`disable_hub_initiated_lpm` driver-opt + balanced enable** — defense against per-LPM-induced data-corruption on quirky devices.
- **Per-`needs_altsetting0` deferred re-install on unbind** — defense against per-orphan alt-setting state across rebind.
- **Per-`pm_runtime_set_active` + `pm_suspend_ignore_children(false)` at claim/probe** — defense against per-runtime-PM-deadlock with hub.
- **Per-bus `need_parent_lock = true` (usb_bus_type)** — defense against per-hub-vs-child race during probe/suspend.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/usb/core/usb.c` (usb_init / usb_disabled / usb_bus init — covered in `core-init.md` Tier-3 if expanded)
- `drivers/usb/core/hub.c` (covered in `core-hub.md` Tier-3)
- `drivers/usb/core/urb.c` (covered in `core-urb.md` Tier-3)
- `drivers/usb/core/devio.c` (covered in `core-devio.md` Tier-3 — usbfs UAPI)
- `drivers/usb/core/generic.c` (usb_generic_driver itself — covered in `core-enumerate.md` Tier-3)
- `drivers/usb/core/message.c` (sync control transfers — covered separately if expanded)
- `drivers/usb/core/config.c` (config/interface descriptor parsing — covered in `core-enumerate.md`)
- `drivers/usb/core/sysfs.c` (sysfs attribute groups — covered separately if expanded)
- USB Type-C / Power-Delivery (`drivers/usb/typec/` — Tier-2 separate)
- USB gadget framework (`drivers/usb/gadget/` — Tier-2 separate)
- LDM `driver_register` / `device_attach` core (covered in `drivers/base/00-overview.md`)
- Implementation code
