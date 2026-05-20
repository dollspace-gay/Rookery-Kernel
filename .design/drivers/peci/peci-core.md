# Tier-3: drivers/peci/{core,request,sysfs}.c — PECI bus framework (Platform Environment Control Interface)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/peci/core.c
  - drivers/peci/request.c
  - drivers/peci/sysfs.c
  - drivers/peci/internal.h
  - include/linux/peci.h
-->

## Summary

The PECI (Platform Environment Control Interface) framework — Intel's single-wire, low-bandwidth proprietary bus for thermal/voltage/error-status telemetry on Intel Xeon and Core processors. The host (typically a BMC or PCH-side controller) is the originator; each CPU socket exposes a PECI client at address `0x30..0x37`. The framework owns: bus_type, per-controller `peci_controller` (one MMIO mailbox-style transport), per-device `peci_device` (one per CPU socket), per-request `peci_request` (fixed 32-byte TX + 32-byte RX buffers), command dispatch (GetDIB, GetTemp, RdPkgConfig/WrPkgConfig, RdIAMSR/WrIAMSR, RdPCIConfig families, RdEndPointConfig family), CPU vendor-family-model matching, retry-on-EAGAIN with exponential backoff up to 700 ms, and per-bus rescan sysfs surface.

This Tier-3 covers `core.c` (~235 lines: bus_type, controller alloc/add, device scan, bus_match by `x86_vfm`, probe/remove dispatch), `request.c` (~482 lines: per-request alloc/xfer/retry, PECI completion-code → -errno translation, per-command builders for DIB/Temp/PkgConfig/PCIConfig/IAMSR/EndPointConfig), and `sysfs.c` (~82 lines: bus-level `rescan` write attribute, per-device `remove` self-remove attribute).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct peci_controller` | per-controller (single mutex `bus_lock`, ops, id) | `drivers::peci::Controller` |
| `struct peci_device` | per-CPU-socket client (`info.x86_vfm`, `info.peci_revision`, `info.socket_id`, `addr`, `deleted`) | `drivers::peci::PeciDevice` |
| `struct peci_request` | per-MBX request (tx.buf[32]/tx.len, rx.buf[32]/rx.len) | `drivers::peci::Request` |
| `struct peci_controller_ops` | `xfer(controller, addr, req)` per-controller op | `drivers::peci::ControllerOps` |
| `struct peci_driver` | per-driver bind (id_table by `x86_vfm`) | `drivers::peci::PeciDriver` |
| `devm_peci_controller_add(parent, ops)` | controller register with auto-cleanup | `Controller::devm_add` |
| `peci_controller_scan_devices(controller)` | scan addrs [0x30, 0x38) creating present devices | `Controller::scan_devices` |
| `peci_device_create(controller, addr)` / `_destroy(device)` | per-device lifecycle | `PeciDevice::create` / `_destroy` |
| `peci_request_alloc(device, tx_len, rx_len)` / `_free(req)` | per-request lifecycle | `Request::alloc` / `_free` |
| `peci_request_status(req)` | PECI completion code → -errno | `Request::status` |
| `peci_request_xfer(req)` | dispatch under `bus_lock` to `controller->ops->xfer` | `Request::xfer` |
| `peci_request_xfer_retry(req)` | retry on -EAGAIN with backoff up to PECI_RETRY_TIMEOUT (700 ms) | `Request::xfer_retry` |
| `peci_xfer_get_dib(device)` / `_get_temp(device)` | spec built-ins | `Request::get_dib` / `_get_temp` |
| `peci_xfer_pkg_cfg_read{b,w,l,q}(device, index, param)` | RDPKGCFG 0xA1 builder family | `Request::pkg_cfg_read*` |
| `peci_xfer_pci_cfg_local_read{b,w,l}` / `peci_xfer_ep_pci_cfg_local_read*` / `peci_xfer_ep_pci_cfg_read*` / `peci_xfer_ep_mmio{32,64}_readl` | PCI/MMIO config readers | `Request::pci_cfg_*` / `_ep_*` |
| `peci_request_data_{cc,readb,readw,readl,readq}` | per-payload extractors | `Request::data_read*` |
| `__peci_driver_register(drv, owner, mod_name)` / `peci_driver_unregister(drv)` | client driver register | `PeciDriver::register` / `_unregister` |
| `peci_bus_type` (`match`/`probe`/`remove`) + `peci_bus_groups` (rescan) | bus_type vtable + bus sysfs group | `Subsystem::bus` |
| `peci_controller_type` / `peci_device_type` | device-model types | `Subsystem::ctrl_type` / `_dev_type` |
| `PECI_BASE_ADDR=0x30`, `PECI_DEVICE_NUM_MAX=8`, `PECI_REQUEST_MAX_BUF_SIZE=32`, `PECI_RETRY_TIMEOUT=700ms` | constants | `consts::PECI_*` |

## Compatibility contract

REQ-1: Bus topology — per-controller bus_lock serializes every `xfer`; per-controller hosts up to 8 CPU-socket devices in PECI address range [0x30, 0x38).

REQ-2: Per-request fixed buffers — `PECI_REQUEST_MAX_BUF_SIZE` (=32) bytes each for TX/RX; alloc refuses with WARN_ON_ONCE on oversized lengths.

REQ-3: Per-MBX command opcode allowlist — only the framework-built families fan out into TX: GetDIB(0xF7), GetTemp(0x01), RdPkgConfig(0xA1)/WrPkgConfig(0xA5), RdIAMSR(0xB1)/WrIAMSR(0xB5)/RdIAMSREx(0xD1), RdPCIConfig(0x61)/WrPCIConfig(0x65), RdPCIConfigLocal(0xE1)/WrPCIConfigLocal(0xE5), RdEndPointConfig(0xC1)/WrEndPointConfig(0xC5) with per-family WR/RD length constants.

REQ-4: PECI completion-code translation `peci_request_status`:
- 0x40 SUCCESS → 0
- 0x80/0x81/0x82 (NEED_RETRY/OUT_OF_RESOURCE/UNAVAIL_RESOURCE) → -EAGAIN
- 0x90 INVALID_REQ → -EINVAL
- 0x91/0x93/0x94 (MCA_ERROR/CATASTROPHIC_MCA/FATAL_MCA), 0x98/0x9B/0x9C (parity errors) → -EIO
- unknown CC → WARN_ONCE + -EIO.

REQ-5: Retry policy — `peci_request_xfer_retry` retries on -EAGAIN with exponential backoff `[1 ms, 128 ms]` up to `PECI_RETRY_TIMEOUT = 700 ms`; sets `PECI_RETRY_BIT` (bit 0 of `tx.buf[1]`) on every retry; interruptible by signal (-ERESTARTSYS).

REQ-6: Bus-match — `peci_bus_match_device_id` matches by `x86_vfm` (vendor-family-model integer from CPU ID); driver `id_table` is null-terminated by `x86_vfm == 0`.

REQ-7: Per-controller IDA — `peci_controller_ida` allocates u8-bounded controller id; controller-name `peci-<id>`.

REQ-8: Per-device address range — scan walks `[0x30, 0x30+PECI_DEVICE_NUM_MAX)` creating per-present-CPU devices; per-device probe (`peci_device_create`) queries DIB to confirm presence.

REQ-9: Per-controller pm_runtime — `pm_runtime_no_callbacks` + `pm_suspend_ignore_children(true)` + `pm_runtime_enable` at add; framework opts out of per-controller runtime-PM callbacks (controller drivers manage their own).

REQ-10: Per-device `deleted` flag prevents double-destroy; sysfs `remove` attribute uses `device_remove_file_self` to atomically detach and then destroy.

REQ-11: Bus rescan sysfs — `echo 1 > /sys/bus/peci/rescan` walks every controller and re-runs `peci_controller_scan_devices`.

REQ-12: Symbol namespace — every exported symbol uses `EXPORT_SYMBOL_NS_GPL(..., "PECI")` — out-of-tree consumers must `MODULE_IMPORT_NS("PECI")`.

## Acceptance Criteria

- [ ] AC-1: Intel Xeon platform with `peci-cpu` controller (e.g. `peci-aspeed` BMC or `peci-npcm`) probes successfully; `dmesg | grep peci` shows controller + per-socket device.
- [ ] AC-2: `peci-cpu` driver binds per-socket; `hwmon` exposes per-core temperature derived from `GetTemp`.
- [ ] AC-3: `echo 1 > /sys/bus/peci/rescan` re-discovers sockets after a CPU hotplug-style event.
- [ ] AC-4: `echo 1 > /sys/bus/peci/devices/<dev>/remove` cleanly removes a device.
- [ ] AC-5: PECI request with EAGAIN completion code retries within 700 ms; -ETIMEDOUT after.
- [ ] AC-6: `peci_request_alloc(_, 64, _)` triggers WARN_ON_ONCE + returns NULL.
- [ ] AC-7: PECI MCA-error completion code returns -EIO to the caller.

## Architecture

```
struct Controller {
  dev: Device,                              // type = peci_controller_type
  ops: &'static ControllerOps,
  bus_lock: Mutex<()>,                      // held for the entire xfer
  id: u8,                                   // ida-allocated, "peci-<id>"
}

struct PeciDevice {
  dev: Device,                              // type = peci_device_type
  info: { x86_vfm: u32, peci_revision: u8, socket_id: u8 },
  addr: u8,                                 // 0x30..0x37
  deleted: bool,
}

struct Request {
  device: &PeciDevice,
  tx: { buf: [u8; 32], len: u8 },
  rx: { buf: [u8; 32], len: u8 },
}

struct ControllerOps { xfer: fn(&Controller, u8 addr, &mut Request) -> i32 }
```

Controller add `devm_peci_controller_add(parent, ops)`:
1. Validate `ops->xfer` non-NULL.
2. Allocate controller; allocate IDA id (u8 max).
3. Set `dev.parent = parent`, `dev.bus = &peci_bus_type`, `dev.type = &peci_controller_type`.
4. Initialize `bus_lock` mutex.
5. `pm_runtime_no_callbacks` + `pm_suspend_ignore_children(true)` + `pm_runtime_enable`.
6. `device_add` then `devm_add_action_or_reset(unregister_controller)`.
7. `peci_controller_scan_devices(controller)` (best-effort; failures non-critical).

Per-controller scan `peci_controller_scan_devices`:
- For addr in `[0x30, 0x30+8)`: `peci_device_create(controller, addr)` (which sends GetDIB to confirm presence).

Per-request xfer `peci_request_xfer(req)`:
1. Grab `to_peci_controller(device->dev.parent)`.
2. `mutex_lock(&controller->bus_lock)`.
3. `controller->ops->xfer(controller, device->addr, req)`.
4. `mutex_unlock`.

Per-request retry `peci_request_xfer_retry(req)`:
1. `WARN_ON(req->tx.len == 0)` (ping not supported).
2. Loop until `time_before(jiffies, start + PECI_RETRY_TIMEOUT)`:
   - `peci_request_xfer(req)`; bail on hard error.
   - If `peci_request_status(req) != -EAGAIN` → return 0.
   - Set `tx.buf[1] |= PECI_RETRY_BIT`.
   - `schedule_timeout_interruptible(wait_interval)`; -ERESTARTSYS on signal.
   - Double wait_interval, capped at `PECI_RETRY_INTERVAL_MAX = 128 ms`.
3. Return -ETIMEDOUT.

Per-driver match `peci_bus_device_match(dev, drv)`:
1. Reject if `dev->type != &peci_device_type`.
2. Walk `id_table` until `x86_vfm == 0`; match on equality.

Sysfs `rescan_store` → `bus_for_each_dev(rescan_controller)` → re-`peci_controller_scan_devices` per controller.

Sysfs per-device `remove_store` → `device_remove_file_self` + `peci_device_destroy` (idempotent via `deleted` flag).

## Hardening

(Inherits row-1 features from `drivers/bus/00-overview.md` § Hardening.)

peci-core specific reinforcement:

- **Fixed 32-byte TX/RX buffers** with `WARN_ON_ONCE` rejection of oversize — defeats per-controller buffer-overflow vector.
- **`bus_lock` mutex over `xfer`** — defeats per-controller multi-CPU concurrent xfer corruption.
- **Per-completion-code translation table** — defeats raw CC propagation to caller.
- **Retry-bit set on every retry** — preserves PECI spec semantic + lets the responder detect duplicate.
- **`deleted` flag** prevents per-device double-destroy across sysfs `remove` + controller-teardown races.
- **`peci_device_create` failure during scan is non-fatal** — controller still usable for present sockets.
- **EXPORT_SYMBOL_NS_GPL("PECI")** — limits out-of-tree consumers to those explicitly importing the namespace.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `peci_controller`, `peci_device`, and `peci_request` (with embedded 32-byte TX/RX); no userland write surface in framework (per-device hwmon attrs live in client drivers, not framework).
- **PAX_KERNEXEC** — PECI framework core in W^X kernel text; `peci_bus_type`, `peci_controller_ops`, `peci_device_type`, and per-request builder tables live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `peci_request_xfer`, `peci_request_xfer_retry`, `peci_request_alloc`, and `peci_controller_scan_devices`.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-controller / per-device `dev` refcount + IDA slot reuse; overflow trap defeats scan/destroy race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `peci_request` (so stale 32-byte RX payloads of one request never bleed into the next), `peci_device`, and `peci_controller`.
- **PAX_UDEREF** — SMAP/PAN enforced at sysfs `rescan_store` and `remove_store` userland entries; `kstrtobool` strictly bounds input parsing.
- **PAX_RAP / kCFI** — `peci_controller_ops::xfer`, `peci_driver` ops (`probe`/`remove`), and `bus_type` ops marked `__ro_after_init` with kCFI-typed indirect dispatch; per-request `device->dev.parent` walked only via `to_peci_controller` container_of.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-controller MMIO base disclosure behind CAP_SYSLOG; suppress `%p` in `peci_request_xfer` debug logs.
- **GRKERNSEC_DMESG** — restrict CC-error, retry-timeout, and unknown-CC banners (`WARN_ONCE`) to CAP_SYSLOG so attackers cannot probe CPU thermal/MCA state via dmesg.
- **MBX cmd allowlist** — `peci_request` `tx.buf[0]` opcode set exclusively by framework builders (GetDIB / GetTemp / RdPkgCfg{b,w,l,q} / WrPkgCfg / RdIAMSR / RdIAMSREx / RdPCICfg{,Local} / WrPCICfg{,Local} / RdEndPointCfg{,_pci,_mmio32,_mmio64} / WrEndPointCfg); no out-of-tree caller-supplied opcode reaches the wire.
- **AW/FCS validation** — per-builder `tx.len` and `rx.len` constants strictly match Intel PECI spec; framework refuses to alloc beyond `PECI_REQUEST_MAX_BUF_SIZE` (32 bytes) via `WARN_ON_ONCE`, so wire-format checksum/length fields cannot be coerced by callers.
- **`/dev/peci-*` CAP_SYS_RAWIO** — any future userland char-device exposure of raw PECI MBX must require CAP_SYS_RAWIO; this framework currently exposes no `/dev/peci-*`, but the contract is preserved for out-of-tree raw-IO consumers.
- **Response-buf bound** — `rx.buf[PECI_REQUEST_MAX_BUF_SIZE]` is a fixed-size in-struct array; per-extractor (`data_readb/w/l/q`) consumes exactly its width starting at offset 1 (after CC byte); -EIO on CC mismatch defeats short-read mis-extraction.
- **Sysfs CAP_SYS_ADMIN** — `/sys/bus/peci/rescan` and per-device `remove` are `0200` (write-only) sysfs files; sysfs default policy restricts to CAP_SYS_ADMIN-capable writers (root in init userns).
- **Retry bound** — `PECI_RETRY_TIMEOUT = 700 ms` caps cumulative retry time; per-iter backoff doubles to `PECI_RETRY_INTERVAL_MAX = 128 ms`; `schedule_timeout_interruptible` allows signal-cancel, defeating PECI-storm soft-lockup.
- **Namespace EXPORT_SYMBOL_NS_GPL** — out-of-tree drivers must `MODULE_IMPORT_NS("PECI")` to link against `peci_xfer_*`, defeating opportunistic raw-MBX abuse from generic modules.

Rationale: PECI exposes thermal, MCA, and configuration state of Intel CPUs to BMC/PCH-side consumers; attacker-side access to PECI lets a compromised BMC probe firmware state, leak microcode revision, or trigger unintended writes via WrPkgConfig/WrIAMSR. RAP/kCFI on `xfer`, command-opcode allowlisting via framework builders, response-buf bound enforcement, and CAP_SYS_ADMIN on sysfs rescan/remove turn the framework from "MBX pipe" into a structural enforcement boundary over CPU-internal state surfaces.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `req_buf_bound` | OOB | `peci_request_alloc(tx, rx)` returns NULL (with WARN) if `tx > 32 || rx > 32`. |
| `addr_range` | OOB | `peci_controller_scan_devices` walks exactly addrs `0x30..0x37`. |
| `cc_translation_total` | TOTAL | `peci_request_status` returns a defined errno (0/-EAGAIN/-EINVAL/-EIO) for every observed CC byte (with WARN_ONCE on unknown). |
| `retry_bound` | LIVENESS | `peci_request_xfer_retry` returns within ~700 ms when responder is stuck at -EAGAIN. |

### Layer 2: TLA+

`models/peci/retry_backoff.tla` (parent-declared): proves the exponential-backoff retry loop with retry-bit set converges (returns within PECI_RETRY_TIMEOUT) and serializes per-controller via `bus_lock` so concurrent client xfer cannot interleave wire bytes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `peci_request_xfer_retry` post: returns 0 only if a SUCCESS or non-EAGAIN error CC was observed; -ETIMEDOUT preserves -EAGAIN history | `Request::xfer_retry` |
| `peci_device_destroy(dev)` post: `dev->deleted == true`; reentry is a no-op | `PeciDevice::destroy` |
| `bus_lock` held across exactly one `ops->xfer` per `peci_request_xfer` call; never recursive | `Request::xfer` |

### Layer 4: Verus/Creusot functional

GetTemp/GetDIB round-trip: builder → xfer → status → data_read* yields temperature/DIB value bit-equal to wire payload; encoded as Verus invariant chained with controller `xfer` postcondition (returns rx.buf populated with CC at [0] + payload at [1..len]).

## Out of Scope

- Per-controller drivers (`peci-aspeed`, `peci-npcm`) — separate future Tier-3
- `peci-cpu` device driver (hwmon front-end) — separate future Tier-3
- 32-bit-only paths
- Implementation code
