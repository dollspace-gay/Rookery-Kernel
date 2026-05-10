# Tier-3: drivers/usb/core/{usb,hub,port,config}.c — USB enumeration + hub driver + per-device config selection

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/usb/00-overview.md
upstream-paths:
  - drivers/usb/core/usb.c
  - drivers/usb/core/hub.c
  - drivers/usb/core/port.c
  - drivers/usb/core/config.c
  - drivers/usb/core/hub.h
  - include/linux/usb.h
-->

## Summary

USB enumeration is what runs from the moment a device is plugged in (or hot-power-applied) until it appears as `/dev/bus/usb/<bus>/<dev>` + every interface bound to a per-class driver. The bulk lives in `drivers/usb/core/hub.c` (~6500 lines — the hub driver, which detects port-status changes, debounces, powers, addresses, fetches descriptors, selects configuration, registers interfaces). This Tier-3 covers the four central files: `usb.c` (subsystem init + struct usb_device alloc + bus_register), `hub.c` (hub driver — biggest single file in usb-core), `port.c` (per-port mgmt + sysfs `/sys/bus/usb/devices/usb<N>-port<M>/`), `config.c` (per-device descriptor parsing + endpoint binding).

Owns: hub-port state machine (DISCONNECTED → POWERED_OFF → POWERED_ON → CONNECTED → ENABLED → ADDRESSED → CONFIGURED → SUSPENDED), debounce-port (200ms wait for stable connection signal), reset-port (USB-2 vs USB-3 reset semantics), addressing (SetAddress(addr) using addr 0 → assigned addr), fetch GET_DEVICE_DESCRIPTOR (8B then full 18B), fetch GET_CONFIG_DESCRIPTOR (chained: config + per-interface + per-altsetting + per-endpoint descriptors), GET_STRING_DESCRIPTOR (manufacturer + product + serial), config selection (SetConfiguration(default = 1, or quirks-table override), interface registration via `usb_create_sysfs_intf_files` + `device_add(&intf->dev)` triggering per-class driver bind, hub-port runtime PM (auto-suspend after ms-idle), USB-3 LPM (Link Power Management U0/U1/U2/U3), USB-PD authorization gate (per-port `authorized` sysfs).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct usb_device` | per-device control block | `kernel::usb::Device` |
| `struct usb_hub` | per-hub control block | `kernel::usb::Hub` |
| `struct usb_port` | per-port control block | `kernel::usb::Port` |
| `struct usb_interface` | per-interface control block | `kernel::usb::Interface` |
| `struct usb_host_config` | per-config descriptor + interface array | `kernel::usb::HostConfig` |
| `usb_alloc_dev(parent, hcd, port1)` | allocate empty usb_device | `Device::alloc` |
| `usb_disconnect(udev)` | disconnect device + cleanup interfaces | `Device::disconnect` |
| `usb_new_device(udev)` | full registration: assign addr, fetch descs, select config, bind interfaces | `Device::new_device` |
| `hub_event(work)` | hub status-change handler (workqueue-driven) | `Hub::event_work` |
| `hub_port_init(hub, udev, port1, retry, parent_hub_state)` | per-port init: reset + address | `Hub::port_init` |
| `hub_port_connect_change(hub, port1, portstatus, portchange)` | port connect-change handler | `Hub::port_connect_change` |
| `hub_port_debounce(hub, port1, must_be_connected)` | 200ms debounce | `Hub::port_debounce` |
| `hub_port_reset(hub, port1, udev, ...)` | port reset (USB-2 SetPortFeature/PORT_RESET, USB-3 SetPortFeature/BH_PORT_RESET) | `Hub::port_reset` |
| `hub_port_status(hub, port1, &portstatus, &portchange)` | read port status | `Hub::port_status` |
| `usb_get_device_descriptor(udev, size)` | fetch device descriptor | `Device::get_device_descriptor` |
| `usb_parse_configuration(udev, cfgno, config, buffer, size)` | parse config descriptor chain | `HostConfig::parse` |
| `usb_get_configuration(udev)` | fetch all config descriptors | `Device::get_configuration` |
| `usb_set_configuration(udev, configuration)` | SET_CONFIGURATION + interface registration | `Device::set_configuration` |
| `usb_set_device_state(udev, new_state)` | state machine transition | `Device::set_state` |
| `usb_register_intf_driver(driver, ...)` / `_unregister_intf_driver` | per-class driver registration | `InterfaceDriver::register` / `_unregister` |
| `usb_get_intfdata(intf)` / `_set_intfdata` | per-interface drvdata | `Interface::data*` |
| `port_event(hub, port1)` | per-port event dispatch | `Port::event` |
| `usb_authorize_device(udev)` / `_deauthorize_device` | per-device authorization (BadUSB defense) | `Device::authorize` / `_deauthorize` |
| `usb_remote_wakeup(udev)` | remote-wakeup signaling | `Device::remote_wakeup` |
| `hub_port_logical_disconnect(hub, port1)` | logical disconnect (over-current handler) | `Hub::port_logical_disconnect` |

## Compatibility contract

REQ-1: Hub-port state machine: DISCONNECTED → POWERED_OFF → POWERED_ON → CONNECTED (debounced) → ENABLED → ADDRESSED → CONFIGURED → SUSPENDED → CONFIGURED transitions byte-identical to upstream.

REQ-2: Debounce-period default 200ms (HUB_DEBOUNCE_TIMEOUT) preserved; configurable via per-quirk override.

REQ-3: SET_ADDRESS retried up to GET_MAXPACKET_TRIES (3) attempts; on USB-3, BH-port-reset used instead of port-reset.

REQ-4: GET_DEVICE_DESCRIPTOR first 8 bytes (to learn ep0 maxpacket size), then full 18 bytes; if first read fails, retry with smaller maxpacket assumption.

REQ-5: Configuration selection: default config index = 1 (per-spec) unless `usb_choose_configuration` quirk-table override; consumer-supplied `usb_device->actconfig` set to chosen.

REQ-6: Per-interface registration: each interface in chosen config registered as `struct usb_interface` w/ child kobject under `/sys/bus/usb/devices/<dev>:<config>.<intf>/`; per-interface modalias `usb:vXXXXpXXXXdlXXXXdhXXXXdcXXdscXXdpXXicXXiscXXipXXinXX` byte-identical.

REQ-7: Per-port sysfs (`/sys/bus/usb/devices/usb<N>-port<M>/`) byte-identical: `connect_type`, `disable`, `quirks`, `over_current_count`, `state`, `usb3_lpm_permit`, `location`, `authorized`.

REQ-8: Authorization gate: per-port + per-device `authorized` flag; `authorized=0` → no SET_CONFIGURATION → no per-class binding (BadUSB defense).

REQ-9: USB-3 LPM (U0/U1/U2/U3) negotiated via SET_FEATURE/U1_ENABLE/U2_ENABLE; per-link timeout via SET_LINK_TIMEOUT.

REQ-10: Per-bus root hub presents as `usb<N>` device with `idVendor=0x1d6b` (Linux Foundation), `idProduct=0x0001/2/3` (USB-1.1/2.0/3.0), as virtual host-controller representation.

## Acceptance Criteria

- [ ] AC-1: USB keyboard plugged into root hub → enumerate sequence completes < 1s, `/dev/input/event<N>` appears, `lsusb -v` matches upstream baseline.
- [ ] AC-2: USB-3 NVMe enclosure → enumerate at SuperSpeed (5Gb/s), `lsusb -t` shows tree with correct speed.
- [ ] AC-3: Hub-tree stress: chain 4 USB-3 hubs deep (max permitted = 5 incl root), end-device enumerates correctly.
- [ ] AC-4: Hot-plug stress: rapid plug/unplug of USB stick 1000x → no urb leak, no devnode leak (KASAN clean).
- [ ] AC-5: BadUSB authorization test: with `authorized_default=0`, plug USB-storage device → enumerated to ADDRESSED state but no per-class binding; `echo 1 > /sys/bus/usb/devices/<dev>/authorized` → triggers configuration + binding.
- [ ] AC-6: USB-3 LPM test: idle USB-3 device transitions to U2 within link-timeout; data transfer wakes back to U0.
- [ ] AC-7: Over-current test: simulate over-current condition → port-power-off + over_current_count incremented + uevent emitted.
- [ ] AC-8: Per-interface modalias test: composite device (e.g., webcam with audio + video + control intfs) generates 3 distinct interface modaliases.

## Architecture

`Device` lives in `kernel::usb::Device`:

```
struct Device {
  refcount: Refcount,
  parent: Option<Arc<Device>>,
  bus: Arc<HcDriver>,
  state: AtomicU8,             // USB_STATE_NOTATTACHED / _ATTACHED / _POWERED / _RECONNECTING / _UNAUTHENTICATED / _DEFAULT / _ADDRESS / _CONFIGURED / _SUSPENDED
  speed: AtomicU8,              // USB_SPEED_LOW / _FULL / _HIGH / _WIRELESS / _SUPER / _SUPER_PLUS
  devnum: u16,
  port_num: u8,
  level: u8,
  descriptor: KBox<UsbDeviceDescriptor>,
  configurations: KBox<[HostConfig]>,
  actconfig: AtomicPtr<HostConfig>,
  string_langid: AtomicU16,
  manufacturer: KString,
  product: KString,
  serial: KString,
  ep0: KBox<HostEndpoint>,
  ep_in: [AtomicPtr<HostEndpoint>; 16],
  ep_out: [AtomicPtr<HostEndpoint>; 16],
  authorized: AtomicBool,
  do_remote_wakeup: AtomicBool,
  reset_resume: AtomicBool,
  port_is_suspended: AtomicBool,
  hub: Option<KBox<Hub>>,        // Some(_) iff this device is itself a hub
  urbnum: AtomicU64,             // total URBs ever submitted (for /sys/.../urbnum)
  active_duration: AtomicU64,    // ms total active
  connected_duration: AtomicU64, // ms total connected
  dev_kobj: KObject,             // /sys/bus/usb/devices/<bus-port-chain>/
}
```

`Hub` lives in `kernel::usb::Hub`:

```
struct Hub {
  intfdev: Arc<Interface>,
  hdev: Arc<Device>,
  events_recvd: AtomicU64,
  ports: KBox<[Arc<Port>]>,
  status_urb: Arc<Urb>,           // listens on ep1-IN for port-status changes
  buffer: KBox<[u32; 4]>,
  status_change_bits: AtomicU64,  // per-port bit
  event_work: WorkStruct,         // hub_event handler
  resume_root_hub: AtomicBool,
  has_indicators: bool,
  in_reset: AtomicBool,
  ...
}
```

`Port` lives in `kernel::usb::Port`:

```
struct Port {
  refcount: Refcount,
  child: Mutex<Option<Arc<Device>>>, // device attached to this port
  hub: Arc<Hub>,
  port_num: u8,
  is_superspeed: bool,
  port_owner: Mutex<Option<Pid>>,    // claimed via USBDEVFS_CLAIM_PORT
  connect_type: PortConnectType,     // hardwired / hotplug / unused
  authorized: AtomicBool,             // per-port BadUSB authorization
  over_current_count: AtomicU32,
  state: AtomicU32,                  // PORT_DISABLED / _POWERED_OFF / _POWERED_ON / _ENABLED / _SUSPENDED
  port_kobj: KObject,                // /sys/bus/usb/devices/usb<N>-port<M>/
}
```

`Hub::event_work`:
1. Read port-status URB completion buffer.
2. For each port-status-change bit set:
   - `Hub::port_status(port_num, &status, &change)`.
   - If C_PORT_CONNECTION: `Hub::port_connect_change(port_num, status, change)`.
   - If C_PORT_OVER_CURRENT: bump `port.over_current_count`, log, possibly disable port.
   - If C_PORT_RESET: complete pending reset.
3. Re-submit status URB.

`Hub::port_connect_change(port_num, status, change)`:
1. If currently-attached child exists: `Device::disconnect(child)`; `port.child = None`.
2. If status & PORT_CONNECTION:
   - `Hub::port_debounce(port_num, must_be_connected=true)` (200ms).
   - `Hub::port_init(self, &child, port_num, retry=0, parent_state)`:
     - `Device::alloc(parent=self.hdev, hcd=self.bus, port1=port_num)`.
     - `Hub::port_reset(port_num, &child, ...)`:
       - SetPortFeature(PORT_RESET) (USB-2) or SetPortFeature(BH_PORT_RESET) (USB-3).
       - Wait for C_PORT_RESET status-change.
       - Read port status: link-state should now be ENABLED.
     - `Device::set_state(USB_STATE_DEFAULT)`.
     - `usb_get_device_descriptor(child, 8)` (just first 8 bytes for ep0 maxpacket).
     - `Device::set_address(child, devnum)` (SetAddress(devnum) on default address 0).
     - `Device::set_state(USB_STATE_ADDRESS)`.
     - `usb_get_device_descriptor(child, 18)` (full).
   - `Device::new_device(&child)`:
     - `Device::get_configuration(child)` (loop through 1..bNumConfigurations).
     - Per-config: parse config + per-intf + per-altsetting + per-ep descriptors via `HostConfig::parse`.
     - `Device::get_string_descriptor` for manufacturer + product + serial.
     - If `child.authorized`:
       - Choose config via `usb_choose_configuration`.
       - `Device::set_configuration(child, chosen)`:
         - SET_CONFIGURATION control transfer.
         - For each interface in chosen config: `Interface::register(&intf)`:
           - `device_add(&intf.dev)` triggers per-class driver match + bind via `usb_bus_type::probe`.
       - `Device::set_state(USB_STATE_CONFIGURED)`.
     - Generate uevent `add@/devices/.../usb<N>-port<M>/<dev>`.
     - Add to `port.child`.

USB-3 LPM negotiation: on USB-3 device, after CONFIGURED, hub may negotiate U1/U2 link-states via SetPortFeature(U1_TIMEOUT) + SetPortFeature(U2_TIMEOUT). LPM-bus-allowable timeout per hub-tier.

Authorization gate (`Device::set_configuration` checks `child.authorized`): if `!authorized` → skip configuration + interface registration; device sits at ADDRESSED state. Userspace `echo 1 > /sys/bus/usb/devices/<dev>/authorized` triggers `Device::authorize` → re-runs `Device::set_configuration` → triggers binding. This is the BadUSB defense: rogue HID/storage class devices can't claim until user authorizes.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `device_no_uaf` | UAF | `Arc<Device>` outlives all interfaces + URBs; `Device::disconnect` waits for in-flight URBs to complete before drop. |
| `port_state_valid` | TRANSITION | Port state transitions follow USB spec (DISCONNECTED → POWERED → ENABLED → ADDRESSED → CONFIGURED, one-step at a time). |
| `descriptor_parse_no_oob` | OOB | `HostConfig::parse` bounds-checks every wTotalLength + bLength against descriptor end before deref. |
| `addr_no_overflow` | OVERFLOW | devnum allocator (per-bus 1..127 for USB-2, 1..127 for USB-3 [128 = -EBUSY for fast-path future]) bitmap-allocated; bounds-checked. |

### Layer 2: TLA+

`models/usb/hub_state.tla` (parent-declared): proves hub port state machine — DISCONNECTED → POWERED_OFF → POWERED_ON → CONNECTED → ENABLED → SUSPENDED; concurrent disconnect + enumerate + suspend never produce ghost device or stuck-disabled port.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Hub::port_init` post: child device in USB_STATE_ADDRESS or USB_STATE_DEFAULT (depending on retry); port.child set | `Hub::port_init` |
| `Device::new_device` post: state in {USB_STATE_CONFIGURED, USB_STATE_ADDRESS} (CONFIGURED iff authorized); per-intf kobjects all created or fully rolled back | `Device::new_device` |
| `Hub::port_connect_change` invariant: every disconnect of child precedes new-child-init; never two children attached to one port simultaneously | `Hub::port_connect_change` |

### Layer 4: Verus/Creusot functional

`Hub::event_work` ↔ upstream `hub_event` semantic equivalence: for any sequence of port-status-change events, the produced child-device state + sysfs hierarchy + uevent stream matches upstream. Encoded as Verus model: `forall events. rookery_hub_event(events) == upstream_hub_event(events)`.

## Hardening

(Inherits row-1 features from `drivers/usb/00-overview.md` § Hardening.)

enumerate-specific reinforcement:

- **`authorized_default=0` on hot-plug class default-on for HID/storage/Ethernet** (BadUSB defense). Chassis-internal USB devices (laptop touchpad, fingerprint reader) auto-authorized via ACPI _PLD location info.
- **Descriptor parser bounds-checked** at every step (wTotalLength outer bound + per-descriptor bLength inner bound). Defense against malicious device with malformed descriptors causing OOB read.
- **Per-port `connect_type=hardwired`** (chassis-internal) auto-authorized; `connect_type=hotplug` requires user authorization.
- **Per-port over_current_count rate-limited** — sustained over-current → port-power-off + WARN_RATELIMITED to dmesg.
- **GET_STRING_DESCRIPTOR length-clamped** at USB_MAX_STRING_LEN (256 bytes); defense against ridiculously-long string descriptor causing memory blowup.
- **Hub-tree depth cap = 7** (per USB spec); enforced at `Hub::port_init`. Defense against cycle-creating malicious hub.
- **SET_ADDRESS race-free** — per-bus `addr0_busy` lock prevents two devices simultaneously trying to use address 0 during enumeration.
- **Pre-CONFIGURED state interfaces have no per-class driver bound** (CONFIGURED is the only state where binding occurs); defense against pre-config interface-driver poking causing UAF.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-bus host-controller driver (covered in `host-xhci.md`, `host-ehci.md`, etc. future Tier-3s)
- URB submission (covered in `core-urb.md` future Tier-3)
- usbfs `/dev/bus/usb/` chardev (covered in `core-devio.md` future Tier-3)
- Per-driver lifecycle (covered in `core-driver.md` future Tier-3)
- Per-class drivers (storage, serial, hid, etc.)
- 32-bit-only paths
- Implementation code
