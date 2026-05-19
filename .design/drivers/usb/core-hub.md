# Tier-3: drivers/usb/core/hub.c — USB hub device + enumeration

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/usb/00-overview.md
upstream-paths:
  - drivers/usb/core/hub.c (~6567 lines)
  - drivers/usb/core/hub.h
  - include/linux/usb/ch11.h (hub class request + port-status bits)
  - include/linux/usb/hcd.h (struct usb_hcd, hc_driver hooks)
-->

## Summary

The **USB hub driver** is the engine that turns a USB bus into a topology: it polls each hub's status-change endpoint, fans port events out to per-port state machines, runs the device-enumeration handshake (reset → set-address → get-descriptor → configure), and feeds children to `usb_new_device`. Per-hub: `struct usb_hub` (one per USB hub interface, including every root hub) holding the IRQ URB, an `event_bits[]` change mask, an array of `struct usb_port`, a TT (transaction translator) sub-object for USB-2 high-speed hubs that translate to low/full-speed children, and a `struct work_struct events` that runs `hub_event` on `hub_wq`. Per-port: `struct usb_port` with PORT_CONNECTION / PORT_ENABLE / PORT_RESET / PORT_OVER_CURRENT status tracking, runtime PM, peer (USB-2 ↔ USB-3 connector mapping), and per-port owner for hub-port claim. Per-event flow: hub status URB (interrupt IN) completes → `hub_irq` captures the change bitmap into `hub->event_bits[0]` → `kick_hub_wq` queues `hub->events` → `hub_event` walks every flagged port and calls `port_event` → on PORT_C_CONNECTION the path enters `hub_port_connect` → `hub_port_init` (reset, set-address, ep0 max-packet probing, get-descriptor) → `usb_new_device` (configurations, driver match). Per-USB-3 SuperSpeed: enumeration is hot-loop-different (HCD already addresses on connect, SS reset uses hot-reset / warm-reset PORT_LINK_STATE machine). Per-TT: split-transactions are scheduled by HCDs through `usb_hub_clear_tt_buffer`. Per-OTG: dual-role / HNP / SRP detection in `usb_enumerate_device_otg`. Per-quirk: `HUB_QUIRK_DISABLE_AUTOSUSPEND`, `HUB_QUIRK_CHECK_PORT_AUTOSUSPEND`, `HUB_QUIRK_REDUCE_FRAME_INTR_BINTERVAL`. Critical for: every USB device discovery, hot-plug, root-hub bring-up, USB-3 lane training observation, OTG role-swap.

This Tier-3 covers `drivers/usb/core/hub.c` (~6567 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct usb_hub` | per-hub state | `UsbHub` |
| `struct usb_port` | per-port state | `UsbPort` |
| `hub_probe()` | per-interface bind | `Hub::probe` |
| `hub_disconnect()` | per-interface unbind | `Hub::disconnect` |
| `hub_configure()` | per-descriptor parse + alloc | `Hub::configure` |
| `hub_activate()` | per-startup / -resume kick | `Hub::activate` |
| `hub_quiesce()` | per-suspend / -disconnect stop | `Hub::quiesce` |
| `hub_irq()` | per-interrupt-URB completion | `Hub::irq_complete` |
| `hub_event()` | per-workqueue dispatch | `Hub::event_work` |
| `port_event()` | per-port event dispatch | `Hub::port_event` |
| `hub_port_connect()` | per-connect / -disconnect | `Hub::port_connect` |
| `hub_port_connect_change()` | per-port connect-change | `Hub::port_connect_change` |
| `hub_port_init()` | per-device-enumeration | `Hub::port_init` |
| `hub_port_reset()` / `_wait_reset()` | per-reset state machine | `Hub::port_reset` / `Hub::port_wait_reset` |
| `hub_set_address()` | per-SET_ADDRESS | `Hub::set_address` |
| `hub_enable_device()` | per-HCD device-slot enable | `Hub::enable_device` |
| `get_bMaxPacketSize0()` | per-ep0-probe | `Hub::probe_ep0_maxpacket` |
| `hub_port_warm_reset_required()` | per-USB3 SS.Inactive | `Hub::port_warm_reset_required` |
| `hub_tt_work()` | per-TT-buffer-clear work | `Hub::tt_clear_work` |
| `usb_hub_clear_tt_buffer()` | per-TT-clear API | `Hub::clear_tt_buffer` |
| `usb_enumerate_device_otg()` | per-OTG-HNP/SRP | `Hub::enumerate_otg` |
| `usb_new_device()` | per-driver-match | `Hub::new_device` |
| `usb_disconnect()` | per-child teardown | `Hub::disconnect_child` |
| `usb_set_device_state()` | per-udev-state | `Hub::set_device_state` |
| `usb_hub_port_status()` / `hub_ext_port_status()` | per-status query | `Hub::port_status` |
| `usb_kick_hub_wq()` | per-external-wake | `Hub::kick_wq` |
| `hub_handle_remote_wakeup()` | per-wakeup-bit | `Hub::handle_remote_wakeup` |
| `usb_hub_set_port_power()` | per-PORT_POWER | `Hub::set_port_power` |
| `hub_quirk_check_port_auto_suspend` (flag) | per-quirk | `UsbHub.quirks` |

## Compatibility contract

REQ-1: struct usb_hub:
- intfdev: per-interface device pointer.
- hdev: per-`usb_device` for the hub itself.
- kref: per-refcount.
- urb: per-interrupt-IN status-change URB.
- buffer: per-URB receive buffer (`u8[8]`, sized for `USB_MAXCHILDREN ≤ 31`).
- status: per-control-transfer status buffer (union `usb_hub_status` / `usb_port_status`).
- status_mutex: per-status-buffer serialization.
- error / nerrors: per-error counter (after 10 consecutive interrupt errors, queue a hub reset).
- event_bits[1]: per-port status-change bitmap (bit 0 = hub status; bits 1..maxchild = ports).
- change_bits[1]: per-port logical-connect-change bitmap.
- removed_bits[1]: per-port "device explicitly removed" marker (do not re-enumerate).
- wakeup_bits[1]: per-port remote-wakeup-signaled bitmap.
- power_bits[1]: per-port powered bitmap.
- child_usage_bits[1]: per-port child-usage runtime-PM bitmap.
- warm_reset_bits[1]: per-port warm-reset-needed bitmap.
- descriptor: per-hub-class descriptor.
- tt: per-`usb_tt` transaction-translator sub-object.
- mA_per_port: per-port current budget.
- limited_power / quiescing / disconnected / in_reset: per-flag.
- quirk_disable_autosuspend / quirk_check_port_auto_suspend: per-quirk flags.
- has_indicators / indicator[]: per-LED indicator state (per USB 2.0 §11.5.3).
- leds / init_work / post_resume_work / events: per-delayed/regular work_struct.
- irq_urb_lock + irq_urb_retry: per-IRQ-URB resubmit retry timer.
- ports[]: per-port `usb_port *` array of length `hdev->maxchild`.
- onboard_devs: per-onboard-device list.

REQ-2: struct usb_port:
- child: per-attached `usb_device` (NULL = empty).
- dev: per-device-model device (one sysfs `usbX-portY` per port).
- port_owner: per-claim-owner (`usb_hub_claim_port`).
- peer: per-USB-2 ↔ USB-3 connector peer (shared physical jack).
- connector: per-Type-C connector.
- req: per-PM-QoS request (hubs without PORT_POWER control).
- connect_type: per-Hardwired / Hotpluggable / Not-Used.
- state: per-`usb_device_state` mirrored to sysfs.
- state_kn: per-`state` sysfs node.
- location: per-platform-specific physical location.
- status_lock: per-port mutex serializing `port_event` vs `usb_port_{suspend,resume}`.
- over_current_count: per-port OC-event count.
- portnum: per-1-based port number.
- quirks: per-port quirk bits.
- early_stop / ignore_event: per-port skip-event flags.
- is_superspeed: per-cache (`true` for USB-3 root-hub ports).
- usb3_lpm_u1_permit / _u2_permit: per-USB3-LPM permit bits.

REQ-3: hub_probe(intf, id):
- /* Reject malformed hubs */
- if hdev.descriptor.bNumConfigurations > 1 ∨ hdev.actconfig.desc.bNumInterfaces > 1: return -EINVAL.
- /* Topology depth limit */
- if hdev.level == MAX_TOPO_LEVEL: return -E2BIG.
- /* Compile-time external-hub veto */
- if CONFIG_USB_OTG_DISABLE_EXTERNAL_HUB ∧ hdev.parent: return -ENODEV.
- /* Validate hub-class endpoint descriptor */
- if !hub_descriptor_is_sane(desc): return -EIO.
- hub = kzalloc; kref_init.
- INIT_DELAYED_WORK(&hub.leds, led_work); INIT_DELAYED_WORK(&hub.init_work, NULL); INIT_DELAYED_WORK(&hub.post_resume_work, hub_post_resume); INIT_WORK(&hub.events, hub_event).
- usb_get_intf(intf); usb_get_dev(hdev); usb_set_intfdata(intf, hub).
- intf.needs_remote_wakeup = 1; pm_suspend_ignore_children(intf.dev, true).
- /* Apply per-VID/PID quirks from `hub_id_table` driver_info */
- if id.driver_info & HUB_QUIRK_CHECK_PORT_AUTOSUSPEND: hub.quirk_check_port_auto_suspend = 1.
- if id.driver_info & HUB_QUIRK_DISABLE_AUTOSUSPEND: hub.quirk_disable_autosuspend = 1; usb_autopm_get_interface_no_resume(intf).
- if id.driver_info & HUB_QUIRK_REDUCE_FRAME_INTR_BINTERVAL ∧ desc.endpoint[0].desc.bInterval > USB_REDUCE_FRAME_INTR_BINTERVAL: clamp bInterval; usb_set_interface(hdev, 0, 0).
- if hub_configure(hub, &desc.endpoint[0].desc) >= 0: onboard_dev_create_pdevs(hdev, &hub.onboard_devs); return 0.
- hub_disconnect(intf); return -ENODEV.

REQ-4: hub_configure(hub, endpoint):
- Read class descriptor via `get_hub_descriptor`.
- Allocate `hub->ports[]` of length `hdev->maxchild`.
- Allocate per-port `usb_port` device-model objects via `usb_hub_create_port_device`.
- For USB-2 high-speed hubs: `usb_hub_tt_init` (init `hub->tt` with `tt_clear_work`).
- For USB-3 hubs: configure SS-specific fields (per-port `is_superspeed`).
- Allocate `hub->buffer` (8 bytes including babble margin) and `hub->status` union buffer.
- Allocate interrupt-IN URB; pipe = `usb_rcvintpipe(hdev, endpoint->bEndpointAddress)`.
- Power on all ports (`hub_power_on(hub, true)`).
- `hub_activate(hub, HUB_INIT)`: submit interrupt URB; mark every port as needing event processing.

REQ-5: hub_activate(hub, type):
- type ∈ { HUB_INIT, HUB_INIT2, HUB_INIT3, HUB_POST_RESET, HUB_RESUME, HUB_RESET_RESUME }.
- /* Clear any stale port status-change features so the first interrupt URB starts clean */
- For each port: usb_clear_port_feature for C_CONNECTION, C_ENABLE, C_SUSPEND, C_OVER_CURRENT, C_RESET, C_BH_PORT_RESET, C_PORT_LINK_STATE, C_PORT_CONFIG_ERROR.
- Submit interrupt-IN URB (`usb_submit_urb(hub->urb, GFP_NOIO)`).
- For deferred-init (HUB_INIT2 / HUB_INIT3): schedule `init_work` after a delay to coordinate with HCD power-on.
- Bump every port into hub_event by setting bits in `hub->change_bits` / `event_bits`; kick the workqueue.

REQ-6: hub_quiesce(hub, type):
- type ∈ { HUB_PRE_RESET, HUB_DISCONNECT, HUB_SUSPEND }.
- hub.quiescing = 1.
- usb_kill_urb(hub.urb).
- For HUB_DISCONNECT: also kill child devices via `hub_disconnect_children`.
- Flush leds / init_work / post_resume_work / events.
- Stop `irq_urb_retry` timer.

REQ-7: hub_irq(urb) (completion of interrupt-IN status URB):
- switch (urb.status):
  - -ENOENT / -ECONNRESET / -ESHUTDOWN: return (do not resubmit).
  - default (error): if ++hub.nerrors < 10 ∨ hub.error: goto resubmit; else hub.error = status; fallthrough.
  - 0 (success):
    - bits = 0.
    - for i in 0..urb.actual_length: bits |= ((u64)((*hub.buffer)[i])) << (i*8).
    - hub.event_bits[0] = bits.
- hub.nerrors = 0.
- kick_hub_wq(hub) — refcount-up the hub and queue `hub->events`.
- resubmit: hub_resubmit_irq_urb(hub).

REQ-8: kick_hub_wq(hub):
- /* Per-hub_get holds the hub alive while work is queued */
- hub_get(hub).
- if !queue_work(hub_wq, &hub.events): hub_put(hub) (work already queued; drop the get).
- /* Refcount balance: matching hub_put at hub_event's exit */

REQ-9: hub_event(work):
- hub = container_of(work, struct usb_hub, events).
- hdev = hub.hdev; hub_dev = hub.intfdev; intf = to_usb_interface(hub_dev).
- kcov_remote_start_usb(hdev.bus.busnum).
- /* Serialize against `usb_disconnect` */
- usb_lock_device(hdev).
- if hub.disconnected: goto out_hdev_lock.
- if hdev.state == USB_STATE_NOTATTACHED: hub.error = -ENODEV; hub_quiesce(hub, HUB_DISCONNECT); goto out_hdev_lock.
- /* Resume the hub if autosuspended */
- if (ret = usb_autopm_get_interface(intf)) != 0: goto out_hdev_lock.
- if hub.quiescing: goto out_autopm.
- /* Recover from accumulated errors */
- if hub.error: usb_reset_device(hdev); hub.nerrors = 0; hub.error = 0.
- /* Per-port pass */
- for i in 1..=hdev.maxchild:
  - port_dev = hub.ports[i - 1].
  - if test_bit(i, hub.event_bits) ∨ test_bit(i, hub.change_bits) ∨ test_bit(i, hub.wakeup_bits):
    - pm_runtime_get_noresume(&port_dev.dev).
    - pm_runtime_barrier(&port_dev.dev).
    - usb_lock_port(port_dev).
    - port_event(hub, i).
    - usb_unlock_port(port_dev).
    - pm_runtime_put_sync(&port_dev.dev).
- /* Hub-level (bit 0) status: HUB_CHANGE_LOCAL_POWER / HUB_CHANGE_OVERCURRENT */
- if test_and_clear_bit(0, hub.event_bits): hub_hub_status(hub, &hubstatus, &hubchange); handle local-power + over-current.
- out_autopm: usb_autopm_put_interface_no_suspend(intf).
- out_hdev_lock: usb_unlock_device(hdev).
- usb_autopm_put_interface(intf); hub_put(hub) (balances kick_hub_wq).
- kcov_remote_stop().

REQ-10: port_event(hub, port1) (called with `port_dev->status_lock` held):
- port_dev = hub.ports[port1 - 1]; udev = port_dev.child.
- connect_change = test_bit(port1, hub.change_bits); clear_bit(port1, hub.event_bits); clear_bit(port1, hub.wakeup_bits).
- if usb_hub_port_status(hub, port1, &portstatus, &portchange) < 0: return.
- if portchange & USB_PORT_STAT_C_CONNECTION: clear_port_feature(C_CONNECTION); connect_change = 1.
- if portchange & USB_PORT_STAT_C_ENABLE:
  - clear_port_feature(C_ENABLE).
  - /* EM-interference hack: hub spontaneously disabled a still-connected device */
  - if !(portstatus & USB_PORT_STAT_ENABLE) ∧ !connect_change ∧ udev: connect_change = 1.
- if portchange & USB_PORT_STAT_C_OVERCURRENT:
  - port_dev.over_current_count++; port_over_current_notify(port_dev).
  - clear_port_feature(C_OVER_CURRENT).
  - msleep(100); hub_power_on(hub, true); re-read status.
- if portchange & USB_PORT_STAT_C_RESET: clear_port_feature(C_RESET).
- if (portchange & USB_PORT_STAT_C_BH_RESET) ∧ hub_is_superspeed(hdev): clear_port_feature(C_BH_PORT_RESET).
- if portchange & USB_PORT_STAT_C_LINK_STATE: clear_port_feature(C_PORT_LINK_STATE).
- if portchange & USB_PORT_STAT_C_CONFIG_ERROR: clear_port_feature(C_PORT_CONFIG_ERROR).
- /* Powered-off ports do nothing else */
- if !pm_runtime_active(&port_dev.dev): return.
- if port_dev.ignore_event ∧ port_dev.early_stop: return.
- if hub_handle_remote_wakeup(hub, port1, portstatus, portchange): connect_change = 1.
- /* USB-3 SS.Inactive recovery: warm-reset loop */
- while hub_port_warm_reset_required(hub, port1, portstatus):
  - if i++ < DETECT_DISCONNECT_TRIES ∧ udev: msleep(20); re-read status; continue.
  - else if !udev ∨ !(portstatus & USB_PORT_STAT_CONNECTION) ∨ udev.state == USB_STATE_NOTATTACHED:
    - hub_port_reset(hub, port1, NULL, HUB_BH_RESET_TIME, true) (warm-reset port only).
    - if !udev ∧ err == -ENOTCONN: connect_change = 0; else if err < 0: hub_port_disable(hub, port1, 1).
  - else: usb_unlock_port; usb_lock_device(udev); usb_reset_device(udev); usb_unlock_device(udev); usb_lock_port; connect_change = 0.
  - break.
- if connect_change: hub_port_connect_change(hub, port1, portstatus, portchange).

REQ-11: hub_port_connect_change(hub, port1, portstatus, portchange):
- /* Try to resuscitate a still-connected device first */
- if (portstatus & USB_PORT_STAT_CONNECTION) ∧ udev ∧ udev.state != USB_STATE_NOTATTACHED:
  - if portstatus & USB_PORT_STAT_ENABLE:
    - descr = usb_get_device_descriptor(udev).
    - if !descriptors_changed(udev, descr, udev.bos): status = 0 (nothing to do).
  - else if CONFIG_PM ∧ udev.state == USB_STATE_SUSPENDED ∧ udev.persist_enabled: treat as wakeup.
- if status != 0: hub_port_connect(hub, port1, portstatus, portchange).

REQ-12: hub_port_connect(hub, port1, portstatus, portchange):
- /* Disconnect any existing child first */
- if port_dev.child: usb_phy_notify_disconnect(...); usb_disconnect(&port_dev.child).
- /* Clear "removed" marker on any connect-status change */
- if !(portstatus & USB_PORT_STAT_CONNECTION) ∨ (portchange & USB_PORT_STAT_C_CONNECTION): clear_bit(port1, hub.removed_bits).
- /* Debounce: re-read until status is stable */
- if portchange & (C_CONNECTION | C_ENABLE): portstatus = hub_port_debounce_be_stable(hub, port1).
- /* Empty port */
- if !(portstatus & USB_PORT_STAT_CONNECTION) ∨ test_bit(port1, hub.removed_bits):
  - if hub_is_port_power_switchable(hub) ∧ !usb_port_is_power_on(hub, portstatus) ∧ !port_dev.port_owner: set_port_feature(PORT_POWER).
  - return.
- unit_load = hub_is_superspeed(hub.hdev) ? 150 : 100.
- /* Per-PORT_INIT_TRIES retry loop */
- for i in 0..PORT_INIT_TRIES:
  - if hub_port_stop_enumerate(hub, port1, i): status = -ENODEV; break.
  - usb_lock_port(port_dev); mutex_lock(hcd.address0_mutex); retry_locked = true.
  - udev = usb_alloc_dev(hdev, hdev.bus, port1); usb_set_device_state(udev, USB_STATE_POWERED).
  - udev.bus_mA = hub.mA_per_port; udev.level = hdev.level + 1.
  - udev.speed = hub_is_superspeed(hub.hdev) ? USB_SPEED_SUPER : USB_SPEED_UNKNOWN.
  - choose_devnum(udev); if udev.devnum <= 0: status = -ENOTCONN; goto loop.
  - /* THE enumeration handshake */
  - status = hub_port_init(hub, udev, port1, i, NULL).
  - if status < 0: goto loop.
  - mutex_unlock(hcd.address0_mutex); usb_unlock_port(port_dev); retry_locked = false.
  - if udev.quirks & USB_QUIRK_DELAY_INIT: msleep(2000).
  - /* Reject bus-powered child hubs that exceed budget */
  - if udev.descriptor.bDeviceClass == USB_CLASS_HUB ∧ udev.bus_mA <= unit_load:
    - usb_get_std_status(udev, USB_RECIP_DEVICE, 0, &devstat).
    - if !(devstat & (1 << USB_DEVICE_SELF_POWERED)): status = -ENOTCONN; goto loop_disable.
  - if udev.descriptor.bcdUSB >= 0x0200 ∧ udev.speed == USB_SPEED_FULL ∧ highspeed_hubs != 0: check_highspeed(hub, udev, port1).
  - /* Publish the child */
  - mutex_lock(&usb_port_peer_mutex); spin_lock_irq(&device_state_lock).
  - if hdev.state == USB_STATE_NOTATTACHED: status = -ENOTCONN; else port_dev.child = udev.
  - spin_unlock_irq; mutex_unlock.
  - if !status: status = usb_new_device(udev) (driver-match).
  - if !status: usb_phy_notify_connect(...); return.
  - loop_disable: hub_port_disable(hub, port1, 1).
  - loop: usb_ep0_reinit(udev); release_devnum(udev); hub_free_dev(udev); if retry_locked: unlocks; usb_put_dev(udev).
  - if status == -ENOTCONN ∨ status == -ENOTSUPP: break.
  - if i == (PORT_INIT_TRIES - 1) / 2: power-cycle the port.
- if !hub.hdev.parent ∧ hcd.driver.port_handed_over ∧ hcd.driver.port_handed_over(hcd, port1): /* HCD took over (port-companion handoff for EHCI ↔ xHCI) */.
- hub_port_disable(hub, port1, 1).
- if hcd.driver.relinquish_port ∧ !hub.hdev.parent: hcd.driver.relinquish_port(hcd, port1).

REQ-13: hub_port_init(hub, udev, port1, retry_counter, dev_descr):
- buf = kmalloc(GET_DESCRIPTOR_BUFSIZE, GFP_NOIO); if !buf: return -ENOMEM.
- delay = HUB_SHORT_RESET_TIME; if !hdev.parent: delay = HUB_ROOT_RESET_TIME; if oldspeed == USB_SPEED_LOW: delay = HUB_LONG_RESET_TIME.
- /* Reset; may upshift speed (FS→HS or up to SuperSpeed) */
- retval = hub_port_reset(hub, port1, udev, delay, false).
- if oldspeed != UNKNOWN ∧ oldspeed != udev.speed ∧ !(oldspeed == SUPER ∧ udev.speed > oldspeed): goto fail.
- /* ep0 max-packet guess */
- if initial: switch udev.speed { SUPER/SUPER_PLUS: 512; HIGH: 64; FULL: 64 (provisional); LOW: 8; default: goto fail }.
- /* TT (transaction-translator) setup: child of HS hub, but speed < HIGH */
- if initial ∧ udev.speed != USB_SPEED_HIGH ∧ hdev.speed == USB_SPEED_HIGH:
  - if !hub.tt.hub: retval = -EINVAL; goto fail.
  - udev.tt = &hub.tt; udev.ttport = port1.
- /* If hub itself has a TT (multi-TT mode), inherit */
- if hdev.tt: udev.tt = hdev.tt; udev.ttport = hdev.ttport.
- do_new_scheme = use_new_scheme(udev, retry_counter, port_dev).
- for retries in 0..GET_DESCRIPTOR_TRIES:
  - if hub_port_stop_enumerate(hub, port1, retries): retval = -ENODEV; break.
  - /* New scheme (Windows-style): 64-byte GET_DESCRIPTOR before SET_ADDRESS */
  - if do_new_scheme:
    - hub_enable_device(udev) (xHCI: assign device slot).
    - maxp0 = get_bMaxPacketSize0(udev, buf, GET_DESCRIPTOR_BUFSIZE, retries == 0).
    - if maxp0 > 0 ∧ !initial ∧ maxp0 != udev.descriptor.bMaxPacketSize0: retval = -ENODEV; goto fail.
    - hub_port_reset(hub, port1, udev, delay, false).
    - if oldspeed != udev.speed: retval = -ENODEV; goto fail.
    - if maxp0 < 0: continue.
  - /* SET_ADDRESS (up to SET_ADDRESS_TRIES = 2 attempts) */
  - for operations in 0..SET_ADDRESS_TRIES:
    - retval = hub_set_address(udev, devnum).
    - if retval >= 0: break.
    - msleep(200).
  - if retval < 0: goto fail.
  - msleep(10) (let SET_ADDRESS settle).
  - if do_new_scheme: break.
  - /* Old scheme: 8-byte GET_DESCRIPTOR after SET_ADDRESS to learn ep0 maxpacket */
  - maxp0 = get_bMaxPacketSize0(udev, buf, 8, retries == 0).
  - if maxp0 >= 0: usb_set_isoch_delay(udev) (USB-3); break.
- /* Re-init ep0 if guess was wrong */
- i = maxp0; if udev.speed >= USB_SPEED_SUPER: i = (maxp0 <= 16) ? (1 << maxp0) : 0.
- if usb_endpoint_maxp(&udev.ep0.desc) == i: pass.
- else if valid speed/i combination: udev.ep0.desc.wMaxPacketSize = cpu_to_le16(i); usb_ep0_reinit(udev).
- else: retval = -EMSGSIZE; goto fail.
- /* Full device descriptor */
- descr = usb_get_device_descriptor(udev); if IS_ERR(descr): retval = PTR_ERR(descr); goto fail.
- if initial: udev.descriptor = *descr; else: *dev_descr = *descr; kfree(descr).
- /* Re-check SS bcdUSB sanity: ≥ 0x0300 expected */
- if udev.speed >= USB_SPEED_SUPER ∧ le16_to_cpu(udev.descriptor.bcdUSB) < 0x0300: warm-reset; retval = -EINVAL; goto fail.
- usb_detect_quirks(udev).
- /* BOS descriptor (USB 2.01+) → LPM capability */
- if udev.descriptor.bcdUSB >= 0x0201:
  - usb_get_bos_descriptor(udev); udev.lpm_capable = usb_device_supports_lpm(udev); usb_set_lpm_parameters(udev); usb_req_set_sel(udev).
- if hcd.driver.update_device: hcd.driver.update_device(hcd, udev).
- hub_set_initial_usb2_lpm_policy(udev).
- fail: if retval: hub_port_disable(hub, port1, 0); update_devnum(udev, devnum); kfree(buf); return retval.

REQ-14: hub_port_reset (state machine):
- For USB-2: SET_FEATURE PORT_RESET, poll PORT_STAT_RESET clearing, check PORT_STAT_ENABLE + PORT_STAT_CONNECTION; record speed from status bits.
- For USB-3: hot-reset by setting PORT_RESET + observing PORT_LINK_STATE; warm-reset via SET_FEATURE BH_PORT_RESET for SS.Inactive / Compliance recovery.
- HUB_RESET_TIMEOUT / HUB_LONG_RESET_TIME / HUB_BH_RESET_TIME bound the polling.

REQ-15: Transaction-Translator (TT) handling:
- Per-USB-2 high-speed hub with low/full-speed children: `struct usb_tt` embedded in `usb_hub.tt` with `clear_work` (work_struct) + `clear_list` (split-transaction errors).
- usb_hub_clear_tt_buffer(urb): HCD callback when a control / interrupt split-transaction errors; queues a `usb_tt_clear` onto `hub.tt.clear_list` and schedules `hub_tt_work`.
- hub_tt_work: drains `clear_list`, issues `HUB_CLEAR_TT_BUFFER` class request (for control ep both directions), and calls back `hc_driver.clear_tt_buffer_complete`.

REQ-16: OTG / dual-role:
- usb_enumerate_device_otg(udev): on OTG-capable root-hub port, parses OTG descriptor; if HNP-supported, sets `USB_DEVICE_B_HNP_ENABLE` feature, else `USB_DEVICE_A_ALT_HNP_SUPPORT` on legacy devices.
- bus.is_b_host / bus.b_hnp_enable / bus.otg_port mediate role-swap.
- During HNP, port_connect debounce is skipped (`portchange &= ~(C_CONNECTION | C_ENABLE)` in `hub_port_connect_change`).

REQ-17: USB-3 SuperSpeed enumeration variant:
- Reset is largely autonomous: HCD trains the link and reports `USB_PORT_STAT_ENABLE | USB_PORT_STAT_CONNECTION` together.
- SS.Inactive (PORT_LINK_STATE) requires warm reset (`hub_port_warm_reset_required` → `hub_port_reset(..., true)`).
- SET_ADDRESS may be HCD-driven via `hcd.driver.address_device` rather than control transfer.
- usb_req_set_sel: configure SystemExitLatency for U1/U2 LPM.
- usb_set_isoch_delay: SS-only `USB_REQ_SET_ISOCH_DELAY` control transfer.

REQ-18: Hub quirks (`HUB_QUIRK_*`):
- CHECK_PORT_AUTOSUSPEND (BIT 0): inspect port-status before allowing per-port autosuspend.
- DISABLE_AUTOSUSPEND (BIT 1): keep autopm-get reference for the lifetime of the hub.
- REDUCE_FRAME_INTR_BINTERVAL (BIT 2): clamp interrupt bInterval to reduce per-frame interrupt rate.
- Quirks are matched per-VID/PID via the `hub_id_table` driver_info.

## Acceptance Criteria

- [ ] AC-1: Plug a device into a powered hub port: `port_event` observes PORT_C_CONNECTION; `hub_port_connect` runs; child appears in sysfs.
- [ ] AC-2: Pull a device: PORT_C_CONNECTION fires with PORT_CONNECTION=0; `usb_disconnect` tears down child; `port_dev->child = NULL`.
- [ ] AC-3: Hub-class enumeration handshake: hub_port_init issues SET_ADDRESS within SET_ADDRESS_TRIES retries; ep0 maxpacket adopted from descriptor if wrong.
- [ ] AC-4: Full-speed device on a HS hub: udev.tt = &hub.tt; ttport = port1; HS hub schedules split-transactions through TT.
- [ ] AC-5: Over-current on a port: PORT_C_OVERCURRENT fires; over_current_count increments; uevent emitted; hub_power_on retries after cool-down.
- [ ] AC-6: USB-3 SS.Inactive: `hub_port_warm_reset_required` returns true → warm reset via `hub_port_reset(..., warm=true)` recovers the link.
- [ ] AC-7: Interrupt URB error 10× consecutive: hub.error set; on next `hub_event` the hub is reset via `usb_reset_device(hdev)`.
- [ ] AC-8: HUB_QUIRK_DISABLE_AUTOSUSPEND device (matched by VID/PID): autopm-get held; hub never autosuspends.
- [ ] AC-9: PORT_INIT_TRIES exhausted: hub_port_connect logs "unable to enumerate USB device" and disables the port; HCD `relinquish_port` is invoked on root-hub ports.
- [ ] AC-10: OTG-capable root-hub port with HNP-supported device: B_HNP_ENABLE feature set; bus.b_hnp_enable = 1.
- [ ] AC-11: hub_disconnect: hub.disconnected = 1; hub_quiesce kills URB; children disconnect recursively; hub_put balances `kref`.
- [ ] AC-12: kick_hub_wq: hub_get held while work queued; hub_put at end of `hub_event` balances exactly once.
- [ ] AC-13: Topology depth at MAX_TOPO_LEVEL: hub_probe returns -E2BIG.
- [ ] AC-14: hub_port_reset on a removed device: returns -ENOTCONN promptly without DETECT_DISCONNECT_TRIES blowing past the budget.

## Architecture

```
struct UsbHub {
  intfdev: *Device,
  hdev: *UsbDevice,
  kref: KRef,
  urb: *Urb,                          // interrupt-IN status URB
  buffer: [u8; 8],
  status: UnsafeCell<HubOrPortStatus>,
  status_mutex: Mutex<()>,
  error: i32,
  nerrors: i32,
  event_bits: [u64; 1],
  change_bits: [u64; 1],
  removed_bits: [u64; 1],
  wakeup_bits: [u64; 1],
  power_bits: [u64; 1],
  child_usage_bits: [u64; 1],
  warm_reset_bits: [u64; 1],
  descriptor: *UsbHubDescriptor,
  tt: UsbTt,                          // transaction translator
  mA_per_port: u32,
  flags: HubFlags,                    // limited_power | quiescing | disconnected | in_reset | quirk_*
  has_indicators: bool,
  indicator: [u8; USB_MAXCHILDREN],
  leds: DelayedWork,
  init_work: DelayedWork,
  post_resume_work: DelayedWork,
  events: WorkStruct,                 // hub_event
  irq_urb_lock: SpinLock<()>,
  irq_urb_retry: TimerList,
  ports: Vec<*UsbPort>,
  onboard_devs: ListHead,
}

struct UsbPort {
  child: Option<*UsbDevice>,
  dev: Device,
  port_owner: Option<*UsbDevState>,
  peer: Option<*UsbPort>,
  connector: Option<*TypecConnector>,
  req: Option<*DevPmQosRequest>,
  connect_type: UsbPortConnectType,
  state: UsbDeviceState,
  state_kn: *KernfsNode,
  location: UsbPortLocation,
  status_lock: Mutex<()>,
  over_current_count: u32,
  portnum: u8,
  quirks: u32,
  flags: PortFlags,                   // early_stop | ignore_event | is_superspeed | usb3_lpm_u1_permit | _u2_permit
}
```

`Hub::probe(intf, id) -> Result<(), i32>`:
1. Validate single-config / single-interface contract.
2. Topology-depth check (≤ MAX_TOPO_LEVEL).
3. hub = UsbHub::new(); kref_init.
4. Init work items + onboard_devs list + irq_urb_lock + irq_urb_retry timer.
5. Set per-quirk fields from id.driver_info.
6. Clamp REDUCE_FRAME_INTR_BINTERVAL endpoint.
7. Hub::configure(hub, endpoint) → on success, run onboard_dev_create_pdevs + Ok.
8. On failure: hub_disconnect.

`Hub::event_work(hub)`:
1. usb_lock_device(hdev).
2. If disconnected / NOTATTACHED: quiesce + early-return.
3. autopm_get_interface (resume hub).
4. If hub.error: usb_reset_device.
5. For i in 1..=hdev.maxchild with event/change/wakeup bit set:
   - pm_runtime_get_noresume + barrier.
   - usb_lock_port; Hub::port_event(hub, i); usb_unlock_port.
   - pm_runtime_put_sync.
6. Handle bit 0 (hub-level local-power / over-current).
7. autopm_put + unlock_device + hub_put.

`Hub::port_event(hub, port1)`:
1. Read portstatus / portchange via `usb_hub_port_status`.
2. For each C_* status-change bit set: usb_clear_port_feature(matching feat) + record connect_change.
3. Handle over-current cool-down loop.
4. Skip if port runtime-suspended.
5. Handle remote wakeup.
6. SS warm-reset loop until link out of SS.Inactive.
7. If connect_change: Hub::port_connect_change.

`Hub::port_init(hub, udev, port1, retry, &mut dev_descr) -> Result<(), i32>`:
1. Reset port; record speed; reject illegal speed transition.
2. Provisional ep0 maxpacket from speed.
3. Bind TT if FS/LS child of HS hub.
4. Iterate up to GET_DESCRIPTOR_TRIES:
   - If new scheme: enable_device + 64-byte GET_DESCRIPTOR + re-reset.
   - SET_ADDRESS loop (≤ SET_ADDRESS_TRIES).
   - If old scheme: 8-byte GET_DESCRIPTOR.
5. Reconcile ep0 maxpacket with descriptor.
6. Full GET_DESCRIPTOR.
7. SS sanity: bcdUSB ≥ 0x0300.
8. usb_detect_quirks.
9. BOS descriptor + LPM parameters + SET_SEL.
10. hcd.update_device.
11. hub_set_initial_usb2_lpm_policy.

`Hub::port_connect(hub, port1, portstatus, portchange)`:
1. Disconnect any existing child.
2. Debounce.
3. Empty-port short-circuit.
4. PORT_INIT_TRIES retry loop:
   - lock_port + address0_mutex.
   - usb_alloc_dev; assign devnum; mark POWERED.
   - Hub::port_init.
   - Reject bus-powered hub exceeding budget.
   - Publish child (`port_dev.child = udev`).
   - Hub::new_device (driver match).
   - Power-cycle at midpoint of retries.
5. Final: relinquish_port to HCD if root-hub port.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `hub_kref_balanced` | INVARIANT | per-probe/disconnect: kref_init / kref_put balanced; events queued imply held ref. |
| `port_status_lock_held_in_port_event` | INVARIANT | per-`Hub::port_event`: caller holds `port_dev.status_lock` (annotated `__must_hold`). |
| `address0_mutex_held_during_set_address` | INVARIANT | per-`Hub::port_init`: hcd.address0_mutex held across SET_ADDRESS attempts in new-scheme path. |
| `devnum_release_on_fail` | INVARIANT | per-`Hub::port_connect` loop: devnum released + hub_free_dev invoked when status != 0. |
| `event_bits_cleared_on_port_event` | INVARIANT | per-`Hub::port_event`: clear_bit(port1, event_bits) before exiting. |
| `hub_event_lock_balance` | INVARIANT | per-`Hub::event_work`: every usb_lock_device path matches usb_unlock_device. |
| `tt_clear_list_drained_on_quiesce` | INVARIANT | per-`Hub::quiesce`: tt.clear_work flushed (cancel_work_sync). |
| `topology_depth_bounded` | INVARIANT | per-probe: hdev.level + 1 ≤ MAX_TOPO_LEVEL. |
| `irq_urb_resubmit_on_transient_error` | INVARIANT | per-hub_irq: transient errors (nerrors < 10) → resubmit; not on -ENOENT/-ESHUTDOWN. |
| `port_init_retry_bounded` | INVARIANT | per-`Hub::port_connect`: at most PORT_INIT_TRIES iterations. |

### Layer 2: TLA+

`drivers/usb/core-hub.tla`:
- States: Idle, IrqPending, EventScheduled, EventRunning, PortReset, SetAddress, GetDescriptor, NewDevice, Connected, Quiescing, Disconnected.
- Properties:
  - `safety_no_double_address` — per-port: SET_ADDRESS only issued once per `Hub::port_init` invocation (modulo SET_ADDRESS_TRIES retries on error).
  - `safety_one_child_per_port` — per-port: `port_dev.child` is either NULL or one `usb_device`; never two.
  - `safety_disconnect_before_reconnect` — per-PORT_C_CONNECTION when child existed: usb_disconnect runs before new alloc.
  - `safety_warm_reset_terminates` — per-SS.Inactive: warm-reset loop bounded by DETECT_DISCONNECT_TRIES.
  - `safety_address0_mutex_exclusive` — per-bus: at most one hub holds `hcd.address0_mutex` at a time.
  - `liveness_event_eventually_runs` — per-irq with bits set: hub_event eventually executes.
  - `liveness_connect_eventually_resolves` — per-PORT_INIT_TRIES: connect-loop terminates in success / `-ENOTCONN` / hub_port_disable.
  - `liveness_quiesce_drains` — per-`Hub::quiesce`: events / leds / init_work all flushed before return.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Hub::probe` post: hub allocated ∧ urb submitted ∨ -error returned | `Hub::probe` |
| `Hub::event_work` post: every flagged port has been visited; bits cleared | `Hub::event_work` |
| `Hub::port_event` post: PORT_C_* features cleared; event_bits & wakeup_bits cleared for port | `Hub::port_event` |
| `Hub::port_init` post (ret == 0): udev addressed ∧ ep0.maxpacket = descriptor value ∧ udev.descriptor populated | `Hub::port_init` |
| `Hub::port_connect` post (ret == 0): port_dev.child = udev ∧ usb_new_device succeeded | `Hub::port_connect` |
| `Hub::set_address` post (ret == 0): udev.devnum = devnum ∧ udev.state = USB_STATE_ADDRESS | `Hub::set_address` |
| `Hub::tt_clear_work` post: tt.clear_list empty | `Hub::tt_clear_work` |
| `Hub::irq_complete` post: nerrors reset on success ∨ hub.error set on persistent failure | `Hub::irq_complete` |

### Layer 4: Verus/Creusot functional

`Per-IRQ-URB completion → event_bits update → kick_hub_wq → hub_event → port_event → hub_port_connect → hub_port_init (reset, set-address, get-descriptor) → usb_new_device` semantic equivalence: per-Documentation/driver-api/usb/* and USB 2.0 §11.24 / USB 3.2 §10 hub class specs. Per-TT: per-USB 2.0 §11.14. Per-OTG: per-USB OTG/EH 2.0 §6 HNP/SRP.

## Hardening

(Inherits row-1 features from `drivers/usb/00-overview.md` § Hardening.)

Hub-driver reinforcement:

- **Per-`hub.disconnected` short-circuit** — defense against per-event-after-disconnect UAF on `hub.hdev`.
- **Per-port `status_lock` mutex** — defense against per-`port_event` racing concurrent `usb_port_{suspend,resume}` from a child driver.
- **Per-`hcd.address0_mutex`** — defense against per-bus-wide SET_ADDRESS collisions across simultaneously enumerating ports.
- **Per-`MAX_TOPO_LEVEL` depth cap** — defense against per-malicious or per-buggy chained-hub stack overflow.
- **Per-PORT_INIT_TRIES bounded retry** — defense against per-livelock when a device keeps failing enumeration.
- **Per-`removed_bits` sticky flag** — defense against per-userspace-marked-removed device re-enumerating on transient noise.
- **Per-`hub_descriptor_is_sane` validation** — defense against per-malformed hub-class descriptor with absurd `maxchild` / `bNumPorts`.
- **Per-`bDeviceClass == USB_CLASS_HUB` bus-power audit** — defense against per-budget-violating bus-powered hub cascade.
- **Per-nerrors-10 hub-reset escalation** — defense against per-stuck-interrupt-endpoint denial of service.
- **Per-`hub.tt.clear_work` flushed on quiesce** — defense against per-TT-clear-callback firing after HCD freed.
- **Per-`hub_port_warm_reset_required` bounded loop (DETECT_DISCONNECT_TRIES)** — defense against per-SS.Inactive infinite recovery.
- **Per-USB 2.0 single-config single-interface rejection** — defense against per-spec-non-compliant rogue hub.
- **Per-CONFIG_USB_OTG_DISABLE_EXTERNAL_HUB kill-switch** — defense against per-policy external-hub attach on OTG-only platforms.
- **Per-`kcov_remote_start_usb` for fuzzing coverage** — defense against per-hidden-path fuzzer blind spots.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on every USBDEVFS / `/dev/bus/usb/*` ioctl arg copy; descriptor parsing in `hub_configure` rejects oversize `wTotalLength`.
- **PAX_KERNEXEC** — W^X for any HCD firmware blob staged through hub enumeration (no executable mapping of device-supplied descriptor data).
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at every USB chardev/sysfs ioctl entry.
- **PAX_REFCOUNT** — saturating refcount on `usb_hub`, `usb_port`, `usb_device`, `usb_tt`, per-port `kobject`.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `hub.event_bits[]`, descriptor cache (`hub_descriptor`), per-port status buffers, control-URB transfer buffers.
- **PAX_UDEREF / PAX_MEMORY_UDEREF** — strict user-pointer access for USBDEVFS bulk/ctrl/iso transfer descriptors; guard against guest-supplied DMA targets from `usbip` userspace.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `usb_driver`, `hc_driver`, `usb_device_driver`, hub `urb->complete` callback, `dev_pm_ops`.
- **GRKERNSEC_IO** — driver never calls `iopl/ioperm`; HCD MMIO via `request_mem_region` only.
- **GRKERNSEC_HIDESYM** — `/sys/bus/usb/devices/*/` and `/proc/bus/usb` kernel pointers (`urb`, `hcd`, `udev`) masked from non-CAP_SYSLOG.
- **GRKERNSEC_DMESG** — over-current, port-reset-storm, and HCD-fatal log lines restricted to CAP_SYSLOG.
- **GRKERNSEC_TPE** — Trusted Path Execution for `/dev/bus/usb/*` raw chardev (USBDEVFS), `/dev/usbmon*`, `/dev/usb/hiddev*`.
- **GRKERNSEC_KMOD** — USB class-driver auto-load (mass-storage, HID, CDC-ACM, etc.) gated by CAP_SYS_MODULE; defense against BadUSB / USB-Rubber-Ducky class-binding attacks.
- **GRKERNSEC_MODHARDEN** — module signature required for `usbcore`, `xhci_hcd`, `ehci_hcd`, all class drivers.
- **GRKERNSEC_NO_FBSPLASH** — applies to USB-display devices (DisplayLink) — raw framebuffer access gated.
- **CAP_SYS_RAWIO** strict for USBDEVFS_RESET, USBDEVFS_RESETEP, USBDEVFS_CLEAR_HALT, USBDEVFS_CONNECT, USBDEVFS_DISCONNECT, USBDEVFS_SUBMITURB on isoc/interrupt endpoints.
- **GRKERNSEC_BRUTE** — repeated enumeration failures from a single port (`PORT_INIT_TRIES` exhausted) trigger per-port quarantine + audit log.
- **USBGUARD-integration hook** — `hub_port_init` calls into LSM hook to consult userspace USBGuard policy before completing `usb_new_device`; defense against BadUSB attaching unauthorized class.
- **OTG-external-hub kill-switch** — `CONFIG_USB_OTG_DISABLE_EXTERNAL_HUB` enforced at runtime via grsec policy on locked-down devices.

Per-driver rationale: the USB hub driver is the primary attack surface for physical-USB threats (BadUSB, malicious HID injection, USB-killer-style power events, malformed-descriptor stack-smashing) and for USBIP / VFIO-USB remote-attached devices that effectively let an attacker present arbitrary descriptors to the kernel; grsec reinforces by sanitizing descriptor buffers on free, signing every hub/HCD/class-driver ops vtable (kCFI), gating raw USBDEVFS access behind TPE+CAP_SYS_RAWIO, gating class-driver auto-load (so a plugged-in device cannot force `cdc_acm` / `usb_storage` to load on a locked-down kiosk), and integrating with USBGuard so per-port authorization is enforced before any class driver sees the device.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/usb/core/urb.c URB allocation + submission (covered in `core-urb.md` Tier-3)
- drivers/usb/core/devio.c usbfs ioctls (covered in `core-devio.md` Tier-3 when expanded)
- drivers/usb/host/ehci-hub.c / xhci-hub.c root-hub HCD specifics (covered in `host-ehci.md` / `host-xhci.md` Tier-3)
- drivers/usb/core/quirks.c per-device quirks table (covered in `core-quirks.md` Tier-3 when expanded)
- drivers/usb/core/port.c sysfs port-device model (covered in `core-port.md` Tier-3 when expanded)
- drivers/usb/typec/* Type-C connector orientation (covered separately)
- Implementation code
