# Tier-3: drivers/greybus/{core,manifest,svc,control}.c — Greybus bus subsystem (Project Ara modular phone substrate)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/greybus/core.c
  - drivers/greybus/manifest.c
  - drivers/greybus/svc.c
  - drivers/greybus/control.c
  - drivers/greybus/hd.c
  - drivers/greybus/interface.c
  - drivers/greybus/bundle.c
  - drivers/greybus/connection.c
  - drivers/greybus/operation.c
  - include/linux/greybus.h
-->

## Summary

Greybus is the kernel bus subsystem inherited from Google's Project Ara modular phone — a UniPro/MIPI-based fabric over which an Application Processor (AP) talks to hot-pluggable hardware Modules. Although Ara never shipped, Greybus survives as the substrate for the BeaglePlay (`gb-beagleplay`) and similar SBC/USB-bridged module experiments; the protocol covers every common module class (battery, vibrator, audio, display, sensor, GPIO, I2C, SPI, UART, USB, loopback, raw, firmware mgmt).

The architecture is layered: per-system `gb_host_device` (HD) instantiated by a low-level driver (USB bridge `es2.c`, MIPS beagle bridge `gb-beagleplay.c`); HD links to a Switch/SVC (System Validation Coprocessor) — an in-fabric supervisor that performs module enumeration, power-domain control, route-table mgmt; per-attached Module exposes one or more Interfaces (`gb_interface`), each enumerated by parsing a Manifest blob fetched over the Control CPort (cport 0). Manifests decompose each interface into Bundles (functional units) and CPorts (connections per protocol); per-bundle bind to `greybus_driver` clients (gpio, i2c, battery, etc.) via the `greybus_bus_type` device-model bus.

This Tier-3 covers `core.c` (~380 lines: bus type, driver register, match), `manifest.c` (~530 lines: TLV manifest parser), `svc.c` (~1400 lines: SVC protocol — DME, link, ping, module reset, power, watchdog, intf_eject, hot-unplug), and `control.c` (~580 lines: per-Interface Control connection — manifest size/get, connect/disconnect, mode-switch, bundle PM).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `greybus_bus_type` | device-model bus | `drivers::greybus::Bus` |
| `struct gb_host_device` | per-host (PHY-link) control block | `drivers::greybus::HostDevice` |
| `struct gb_interface` | per-Interface (module port) record | `drivers::greybus::Interface` |
| `struct gb_bundle` | per-Bundle (functional unit) record | `drivers::greybus::Bundle` |
| `struct gb_connection` | per-CPort connection | `drivers::greybus::Connection` |
| `struct gb_operation` | per-request operation (req+resp pair) | `drivers::greybus::Operation` |
| `struct gb_svc` | per-SVC supervisor record | `drivers::greybus::Svc` |
| `struct gb_control` | per-Interface Control protocol client | `drivers::greybus::Control` |
| `greybus_register_driver(drv, owner, mod_name)` / `_deregister_driver(drv)` | bundle-class driver register | `Driver::register` / `_deregister` |
| `gb_match_device(dev, drv)` (in `core.c::greybus_match_device`) | id-table match | `Bus::match` |
| `gb_manifest_parse(intf, data, sz)` | TLV manifest validator | `Interface::parse_manifest` |
| `gb_control_get_manifest_size_operation(intf)` / `_get_manifest_operation(...)` | fetch manifest over CPort 0 | `Control::get_manifest_*` |
| `gb_control_connected_operation(ctrl, cport)` / `_disconnected_operation(...)` / `_disconnecting_operation(...)` | per-cport lifecycle | `Control::connect_*` |
| `gb_control_bundle_{suspend,resume,deactivate,activate}_operation(ctrl, bundle_id)` | bundle PM | `Control::bundle_pm_*` |
| `gb_control_mode_switch_operation(ctrl)` | request module mode-switch | `Control::mode_switch` |
| `gb_svc_intf_device_id(svc, intf, did)` / `_intf_eject(svc, intf)` / `_intf_vsys_set(...)` / `_intf_refclk_set(...)` | SVC supervisory ops | `Svc::intf_*` |
| `gb_svc_dme_peer_get(svc, ...)` / `_dme_peer_set(...)` | UniPro DME peer query | `Svc::dme_peer_*` |
| `gb_svc_route_create(svc, intf1, intf2, dev_id)` / `_route_destroy(svc, ...)` | switch routing table | `Svc::route_create` / `_destroy` |
| `gb_svc_ping_operation(svc)` | heartbeat (watchdog) | `Svc::ping` |
| `gb_svc_pwrmon_*` | power monitor | `Svc::pwrmon_*` |
| `gb_operation_create(...)` / `_request_send_sync(...)` / `_response_send(...)` | request/response engine | `Operation::create` / `send_sync` / `send_response` |

## Compatibility contract

REQ-1: ABI `include/linux/greybus.h` UAPI-adjacent + `include/linux/greybus/` protocol headers must keep on-wire layout stable; per-protocol operation-type byte and request/response struct fields exactly match the Greybus Specification revision in `Documentation/staging/greybus/`.

REQ-2: Manifest format — 4-byte `greybus_manifest_header { size:LE16, version_major:U8, version_minor:U8 }` followed by TLV descriptors (`__le16 size, u8 type, u8 pad` then payload). Total declared `size` must equal payload size copied from the host.

REQ-3: Manifest descriptor types: `INTERFACE` (vendor/product string ids), `STRING` (length-prefixed), `BUNDLE` (id + class), `CPORT` (id + bundle_id + protocol), `MIKROBUS` (BeagleBone-specific). Exactly one `INTERFACE` descriptor required; bundle ids unique within an interface; cport ids unique within a bundle.

REQ-4: CPort 0 is always the Control protocol; reserved by core, instantiated as `intf->control`. Connect/disconnect of non-zero cports flows through the Control CPort.

REQ-5: SVC is itself an Interface at `GB_SVC_INTF_ID = 0`; HD low-level driver creates the SVC connection at probe and routes module hotplug events (`GB_SVC_TYPE_INTF_HOTPLUG`, `_INTF_HOT_UNPLUG`, `_INTF_RESET`) through it.

REQ-6: Per-bundle PM via Control bundle_{suspend,resume,deactivate,activate} maps Greybus status codes (`GB_CONTROL_BUNDLE_PM_*`) into `0`/`-EBUSY`/`-ENOMSG`/`-EINVAL`/`-ENOMEM`/`-EPROTONOSUPPORT` via `gb_control_bundle_pm_status_map`.

REQ-7: SVC watchdog (`svc_watchdog.c`) pings the SVC every N seconds (default 10s); 3 missed responses trigger HD reset.

REQ-8: SVC ECDH/hash-based interface authentication: `intf->ddbl1_manufacturer_id` + `_product_id` + Greybus manifest must match the in-kernel module-authentication policy (when CONFIG_GREYBUS_ES2_AP_AUTH built — currently optional).

REQ-9: Module hot-unplug is asynchronous; per-bundle `greybus_driver.disconnect` callback runs in workqueue context; in-flight operations on dying interface complete with `-ENODEV`.

REQ-10: Per-CPort message size bounded by HD-advertised `hd->buffer_size_max` (typically 2 KiB on ES2 USB AP-Bridge); operations larger than that must use multi-fragment payload at protocol layer.

REQ-11: Per-host module hotplug requires CAP_SYS_ADMIN at the bridge driver layer (`/sys/bus/greybus/devices/...` writes for forced enumeration).

## Acceptance Criteria

- [ ] AC-1: Boot with BeaglePlay + Mikrobus click board; SVC initialised, manifest parsed, bundle driver bound (`/sys/bus/greybus/devices/1-2.0/` populated).
- [ ] AC-2: Manifest fuzz: 1000 randomly-mangled manifests fed via `gb_manifest_parse` — every reject returns a defined error code, never UAF / OOB.
- [ ] AC-3: Hot-unplug of a connected module: per-bundle `disconnect` fires, in-flight ops drain with `-ENODEV`, sysfs nodes removed.
- [ ] AC-4: SVC watchdog forced timeout: HD reset path executes, modules re-enumerate.
- [ ] AC-5: Bundle PM cycle: suspend → resume round-trip on every PM-aware bundle class.
- [ ] AC-6: ES2 USB bridge loopback CPort smoke: 100 k ping-pong operations, no leak in `gb_operation` slab.
- [ ] AC-7: CPort id collision in manifest: rejected with `-EINVAL`, no partial bundle creation.
- [ ] AC-8: Bundle disconnect-while-suspended path: resume + then disconnect; no refcount leak on `gb_interface`.

## Architecture

`Interface` lives in `drivers::greybus::Interface`:

```
struct Interface {
  module: Arc<Module>,
  hd: Arc<HostDevice>,
  control: Option<Arc<Control>>,
  bundles: Mutex<Vec<Arc<Bundle>>>,
  manifest_descs: Mutex<LinkedList<ManifestDesc>>,
  vendor_string: Option<KString>,
  product_string: Option<KString>,
  ddbl1_manufacturer_id: u32,
  ddbl1_product_id: u32,
  interface_id: u8,
  device_id: u8,
  features: u32,
  quirks: u32,
  active: AtomicBool,
  enabled: AtomicBool,
  disconnected: AtomicBool,
  ejected: AtomicBool,
  removed: AtomicBool,
  active_route_create: AtomicBool,
  mutex: Mutex<()>,
}

struct Operation {
  connection: NonNull<Connection>,
  id: u16,
  type_: u8,
  request: KBox<Message>,
  response: Option<KBox<Message>>,
  callback: Option<OperationCb>,
  errno: AtomicI32,
  active: AtomicBool,
  flags: u32,
  refcount: Refcount,
  completion: Completion,
  links: ListLinks,
}
```

Module hot-plug sequence (driven by SVC `GB_SVC_TYPE_INTF_HOTPLUG`):
1. `gb_svc_intf_create(svc, intf_id)` → allocate `gb_interface`, bind to `module`.
2. `gb_svc_intf_vsys_set(svc, intf_id, true)` → power up VSys rail.
3. `gb_svc_intf_refclk_set(svc, intf_id, true)` → enable reference clock.
4. `gb_svc_intf_device_id(svc, intf_id, did)` → assign UniPro device id.
5. `gb_svc_route_create(svc, GB_SVC_INTF_ID, intf_id, did)` → install switch routing entry.
6. Create CPort 0 control connection; `gb_control_get_manifest_size_operation` → bounded by `GB_MANIFEST_MAX_SIZE` (64 KiB upper bound).
7. `gb_control_get_manifest_operation` fetches blob; `gb_manifest_parse(intf, blob, size)` validates TLV stream.
8. For each parsed `BUNDLE` desc create `gb_bundle`; for each `CPORT` desc create `gb_connection`; bind bundles to matching `greybus_driver`.

Manifest parse `gb_manifest_parse`:
1. Validate `header.size == size` (caller-provided).
2. Walk TLV stream — for every descriptor:
   - Validate declared `desc->header.size >= sizeof(desc->header) + payload_min`.
   - Validate cumulative offset `<= header.size`.
   - Hand to `identify_descriptor` which switches on `desc->type` and constructs the in-kernel record on a per-interface list.
3. Locate exactly one `INTERFACE` descriptor; reject manifest without it.
4. `gb_manifest_parse_interface` → `gb_manifest_parse_bundles` → per-bundle `gb_manifest_parse_cports`.
5. Reject duplicate bundle/cport ids; reject CPort ids referencing non-existent bundles.
6. On any parse error: `release_manifest_descriptors` frees all descriptors and per-interface state is clean.

Operation send `gb_operation_request_send_sync`:
1. Allocate per-operation `request` + (optional) `response` message buffers from per-HD slab; size bounded by `hd->buffer_size_max`.
2. Assign per-connection operation id (16-bit, monotonic with wrap; collision-avoidance via per-connection id-bitmap).
3. Hand request to `hd->driver->message_send(conn, op->request)`.
4. Wait on `op.completion` (interruptible or non-interruptible per flags); timeout `GB_OPERATION_TIMEOUT_DEFAULT = 1 s` configurable per-op.
5. Response (if expected) parsed from incoming message; `errno` from response header copied to op.

SVC protocol dispatch `gb_svc_request_handler`:
- per-type table: `INTF_HOTPLUG`, `INTF_HOT_UNPLUG`, `INTF_RESET`, `KEY_EVENT`, `PING`, `MODULE_INSERTED`, `MODULE_REMOVED`. Unknown types return `-EINVAL`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `manifest_parse_no_oob` | OOB | walk through TLV stream never reads past `header.size`; per-descriptor payload bounded. |
| `cport_id_unique` | UNIQUENESS | per-interface cport ids unique; per-bundle cport ids unique. |
| `op_id_no_collision` | UNIQUENESS | per-connection operation id allocator bitmap-tracked; reuse only after response completed. |
| `intf_refcount_no_uaf` | UAF | `Arc<Interface>` outlives all bundles/connections; per-connection close decrements before interface free. |

### Layer 2: TLA+

`models/greybus/operation_engine.tla` — proves request/response pairing with timeout cancel + concurrent hot-unplug terminates: every in-flight op either receives response, times out, or aborts with `-ENODEV`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `gb_manifest_parse` post on `Ok`: every bundle has > 0 cports, every cport references an existing bundle, exactly one INTERFACE descriptor consumed | `Interface::parse_manifest` |
| `gb_svc_route_create` post: switch routing table has entry (src,dst,did); destroy removes it | `Svc::route_create` |
| `gb_control_bundle_pm_status_map` total: every defined status maps to a stable errno | `Control::bundle_pm_status_map` |
| Hotplug/unplug symmetry: every hotplug-allocated interface freed on unplug or HD shutdown | `Svc::intf_hotplug`/`_unplug` |

### Layer 4: Verus functional

Modelled hot-plug → manifest-parse → bundle-bind → bundle-unbind → hot-unplug as a state machine; Verus encoding proves the per-interface state machine has no orphan transition reachable from `disconnected = true`.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

greybus-core specific reinforcement:

- **Manifest size hard cap** — `GB_MANIFEST_MAX_SIZE = 64 KiB`; per-descriptor `size >= header + payload_min` enforced.
- **TLV bounds checking on every step** — `manifest_size` validated against caller-passed size before walking; unknown descriptor types skipped (no OOB read).
- **CPort 0 reserved** — protocol-layer code refuses creating non-Control connections on cport 0.
- **Operation id wrap protection** — per-connection id-bitmap forbids id reuse while in-flight.
- **SVC watchdog dampening** — missed-ping count thresholded before HD reset to absorb transient UniPro stalls.
- **Hotplug serialization** — per-HD enumeration mutex prevents two hotplug events racing for the same intf_id.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `gb_host_device`, `gb_interface`, `gb_bundle`, `gb_connection`, `gb_operation`, message buffers, and manifest descriptor records; on-wire copy paths size-validated before `memcpy_to_msg`.
- **PAX_KERNEXEC** — greybus core text + `greybus_bus_type`, file-ops tables, per-driver `id_table` lookups in `__ro_after_init` W^X kernel text; SVC and Control protocol-handler tables marked `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `gb_manifest_parse`, `gb_operation_request_send_sync`, `gb_svc_intf_hotplug_recv`, `gb_control_bundle_*`, and HD message_send/receive callbacks.
- **PAX_REFCOUNT** — saturating `refcount_t` on `gb_interface`, `gb_bundle`, `gb_connection`, `gb_operation`, and `gb_module`; overflow trap defeats hot-unplug-during-op race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for manifest descriptor records, op request/response buffers, and per-bundle/per-connection state on hot-unplug so stale module identity / payload bytes cannot bleed across reinsertion.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs/debugfs write into greybus (intf_eject, vsys_set, refclk_set, watchdog enable/disable, dme_peer_set).
- **PAX_RAP / kCFI** — `greybus_driver` op-vectors (`probe`, `disconnect`, `id_table`), SVC request handler dispatch table, Control protocol dispatch, and per-HD `gb_hd_driver` callbacks marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-HD bridge MMIO/USB-pointer disclosure behind CAP_SYSLOG; suppress `%p` in manifest/parse failure banners and SVC heartbeat tracepoints.
- **GRKERNSEC_DMESG** — restrict SVC ping-failure, manifest-parse-fail, hot-unplug, and bundle-PM-fail banners to CAP_SYSLOG so attackers cannot fingerprint attached modules via dmesg.
- **Manifest parser bounded** — every TLV descriptor's `size` validated against remaining blob length before deref; refuse manifests > 64 KiB and refuse descriptor counts > 256.
- **CPort id PAX_USERCOPY** — `gb_control_connected_request.cport_id` and per-bundle cport-id arrays validated against per-interface allocation bitmap; refuse out-of-range cport ids.
- **SVC message allowlist** — incoming SVC request types matched against fixed dispatch table; unknown types rejected with `-EINVAL` and rate-limited dmesg under CAP_SYSLOG.
- **AP-module auth challenge** — on hot-plug, before manifest fetch, optional CONFIG_GREYBUS_AUTH ECDH challenge-response gates whether the SVC routes the interface; locked-down kernels refuse unauth'd modules.
- **Hot-swap CAP_SYS_ADMIN** — sysfs writes that force interface eject, vsys toggle, refclk toggle, or watchdog override require CAP_SYS_ADMIN in the bridge driver's user namespace.

Rationale: a Greybus module is functionally a hot-pluggable peripheral with the same trust surface as a USB device but with a custom binary manifest, dynamic CPort routing, and per-bundle PM RPCs. Without manifest-bound parsing, CPort-id validation, auth-gated enumeration, RAP/kCFI on per-protocol dispatch tables, refcount-overflow trapping, and CAP_SYS_ADMIN on supervisory sysfs writes, a malicious or buggy module can corrupt parser state, hijack switch routing, or race hot-unplug into a UAF.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-class bundle drivers (gpio, i2c, battery, etc.) — out of scope at this Tier-3.
- HD low-level drivers `es2.c` / `gb-beagleplay.c` — covered in separate Tier-3.
- UniPro / MIPI Layer-1 framing — vendor IP, not in scope.
- `svc_watchdog.c` details (covered in `svc-watchdog.md` future Tier-3).
- 32-bit-only paths
- Implementation code
