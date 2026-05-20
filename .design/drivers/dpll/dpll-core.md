# Tier-3: drivers/dpll/{dpll_core,dpll_netlink}.c — DPLL (Digital PLL) subsystem + netlink ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/dpll/dpll_core.c
  - drivers/dpll/dpll_core.h
  - drivers/dpll/dpll_netlink.c
  - drivers/dpll/dpll_netlink.h
  - drivers/dpll/dpll_nl.c
  - drivers/dpll/dpll_nl.h
  - include/linux/dpll.h
  - include/uapi/linux/dpll.h
-->

## Summary

The DPLL (Digital Phase-Locked-Loop) subsystem provides a generic device + pin model for synchronization equipment found on telco / 5G / PTP / SyncE-class NICs (Intel ice/idpf/i40e, Marvell prestera, NVIDIA mlx5, Microchip zl3073x) and on standalone DPLL ASICs. A `dpll_device` represents a single DPLL chip (PPS/EEC/EEC-PPS); each device has a set of `dpll_pin` references — input pins (sources of a reference clock — GPS / SyncE / external 10 MHz / 1 PPS) and output pins (clock outputs to PHYs, SMA jacks, BNC). User-space (chronyd, ts2phc, linuxptp, sync4ptp) drives all this via a YAML-spec'd generic-netlink family (`dpll`) — DPLL_CMD_DEVICE_{GET,ID_GET,SET} + DPLL_CMD_PIN_{GET,ID_GET,SET}.

This Tier-3 covers `dpll_core.c` (~1150 lines: device + pin registry, reference-counted ownership, netdev binding, fwnode lookup, notifier chain) + `dpll_netlink.c` (~2100 lines: nl_ops handlers, attribute pack/unpack, notification multicast) plus the YAML-generated `dpll_nl.{c,h}` policy tables.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dpll_device` | per-DPLL-chip state (id, type, mode, lock-status, mode-supported mask, clock_id) | `drivers::dpll::Device` |
| `struct dpll_pin` | per-pin state (idx, prop, direction, freq, phase-offset, freq-support) | `drivers::dpll::Pin` |
| `struct dpll_pin_ref` | bridge: per-(device,pin) registration record | `drivers::dpll::PinRef` |
| `struct dpll_device_ops` / `struct dpll_pin_ops` | driver-supplied per-device + per-pin callbacks | `drivers::dpll::{DeviceOps,PinOps}` |
| `struct dpll_pin_properties` | static pin metadata (name, type, capabilities, board-label, package-label, panel-label, phase-range, freq-range) | `drivers::dpll::PinProperties` |
| `dpll_device_get(clock_id, dev_driver_id, module)` / `_put(...)` | refcounted alloc-or-lookup of a DPLL by (clock_id, driver_id) | `Device::get` / `_put` |
| `dpll_device_register(dpll, type, ops, priv)` / `_unregister(...)` | activate driver ownership over a `dpll_device` | `Device::register` / `_unregister` |
| `dpll_pin_get(clock_id, pin_idx, module, prop, ops, priv)` / `_put(...)` | refcounted alloc-or-lookup of a pin | `Pin::get` / `_put` |
| `dpll_pin_register(dpll, pin, ops, priv)` / `_unregister(...)` | attach a pin to a DPLL device | `Pin::register` / `_unregister` |
| `dpll_pin_on_pin_register(parent_pin, child_pin, ops, priv)` | hierarchical pin (mux input → real input pin) registration | `Pin::on_pin_register` |
| `dpll_netdev_pin_set(dev, pin)` / `_clear(dev)` | bind a netdev's port to a DPLL pin (so `ethtool` can report sync state) | `Pin::set_netdev` / `_clear_netdev` |
| `dpll_pin_fwnode_set(pin, fwnode)` / `fwnode_dpll_pin_find(fwnode, type)` | fwnode/DT linkage so generic ptp/synce binders can find pins | `Pin::set_fwnode` / `Pin::find_by_fwnode` |
| `register_dpll_notifier(nb)` / `unregister_dpll_notifier(nb)` | per-event notifier chain (DEVICE_CREATE/_DELETE/_CHANGE, PIN_*) | `Notifier::register` / `_unregister` |
| `dpll_device_change_ntf(dpll)` / `dpll_pin_change_ntf(pin)` | emit DPLL_CMD_DEVICE_CHANGE_NTF / DPLL_CMD_PIN_CHANGE_NTF multicast | `Device::change_ntf` / `Pin::change_ntf` |
| `dpll_genl_family` | generic-netlink family `dpll` with mcast groups `monitor` | `Netlink::GenlFamily` |
| `dpll_msg_add_lock_status(...)` / `_phase_offset(...)` / `_phase_adjust(...)` / `_clock_quality_level(...)` / `_temp(...)` | per-attribute pack helpers | `Netlink::Attrs::*` |
| `dpll_pin_state_set(...)` / `_prio_set(...)` / `_phase_adjust_set(...)` / `_frequency_set(...)` / `_esync_set(...)` | per-attribute set entries | `Netlink::PinSet::*` |
| `dpll_pin_idx_alloc(*idx)` / `_idx_free(idx)` | global per-pin idx via `dpll_pin_idx_ida` | `Pin::Idx::*` |

## Compatibility contract

REQ-1: Generic-netlink family `"dpll"`, version `DPLL_FAMILY_VERSION`, multicast group `"monitor"` for DEVICE_*_NTF / PIN_*_NTF events. Policy generated from `tools/net/ynl/Documentation/specs/dpll.yaml`.

REQ-2: Device commands (`enum dpll_cmd`):
- `DPLL_CMD_DEVICE_ID_GET` — lookup id by (module-name, clock-id, type).
- `DPLL_CMD_DEVICE_GET` — read one DPLL (or dump all).
- `DPLL_CMD_DEVICE_SET` — write mutable attrs (`DPLL_A_MODE`, `DPLL_A_PHASE_OFFSET_MONITOR`).

REQ-3: Pin commands:
- `DPLL_CMD_PIN_ID_GET` — lookup pin id by (module-name, clock-id, board/package/panel-label, pin-type).
- `DPLL_CMD_PIN_GET` — read one pin (or dump).
- `DPLL_CMD_PIN_SET` — set mutable pin attrs (state, prio, phase-adjust, frequency, esync, parent-state).

REQ-4: Device attributes (`enum dpll_a`):
ID, MODULE_NAME, CLOCK_ID, MODE, MODE_SUPPORTED, LOCK_STATUS, LOCK_STATUS_ERROR, TYPE, TEMP, CLOCK_QUALITY_LEVEL, PHASE_OFFSET_MONITOR.

REQ-5: Pin attributes (`enum dpll_a_pin`):
ID, PARENT_ID, MODULE_NAME, CLOCK_ID, BOARD_LABEL, PACKAGE_LABEL, PANEL_LABEL, TYPE, DIRECTION, FREQUENCY, FREQUENCY_SUPPORTED (nested low/high pairs), CAPABILITIES, STATE, PRIO, PHASE_ADJUST, PHASE_ADJUST_MIN/_MAX, PHASE_OFFSET, FRACTIONAL_FREQUENCY_OFFSET, ESYNC_FREQUENCY, ESYNC_FREQUENCY_SUPPORTED, ESYNC_PULSE.

REQ-6: Lock-status state machine (`enum dpll_lock_status`):
UNLOCKED → LOCKED → LOCKED_HO_ACQ → HOLDOVER (per ITU-T G.781). State transitions reported via DEVICE_CHANGE_NTF.

REQ-7: Hierarchical pins — `dpll_pin_on_pin_register(parent_pin, child_pin, ops, priv)` supports muxed input pins (typical on Marvell/MLX devices) where the "real" input feeds through a mux.

REQ-8: Driver ownership — `dpll_device_register` / `_unregister` install per-driver `dpll_device_ops`; pre-registration the DPLL exists but is "invisible" (`dpll_pin_available` returns false).

REQ-9: Netdev linkage — `dpll_netdev_pin_set(dev, pin)` stores a pin pointer on `net_device`; `ethtool -A`/`tc` and per-NIC drivers read it back to report which port owns synchronization input.

REQ-10: fwnode lookup — `dpll_pin_fwnode_set` + `fwnode_dpll_pin_find` lets ptp_clock + synce drivers find a DPLL pin from a DT node, enabling shared-fwnode binding.

REQ-11: Notifier chain (`dpll_notifier_chain`) — kernel consumers (e.g. ptp4l SyncE bridge) register a `notifier_block` and receive DPLL_*_CREATE/_DELETE/_CHANGE callbacks alongside the netlink multicast.

REQ-12: Module symbol surface (EXPORT_SYMBOL_GPL): `dpll_device_get/_put/_register/_unregister`, `dpll_pin_get/_put/_register/_unregister`, `dpll_pin_on_pin_register/_unregister`, `dpll_netdev_pin_set/_clear`, `__dpll_pin_change_ntf`, `dpll_pin_change_ntf`, `dpll_device_change_ntf`, `dpll_pin_fwnode_set`, `fwnode_dpll_pin_find`, `register_dpll_notifier`, `unregister_dpll_notifier`.

## Acceptance Criteria

- [ ] AC-1: With `ice` driver loaded on E810-XXV-2 NIC, `ynl --family dpll --do device-get --json {}` returns 1+ DPLL entries with lock-status fields.
- [ ] AC-2: `ynl --family dpll --dump pin-get --json '{}'` enumerates all pins (input + output) with board/package/panel labels.
- [ ] AC-3: `ynl --family dpll --do pin-set --json '{"id":<sma1>,"state":"connected"}'` succeeds (CAP_NET_ADMIN required); `pin-get` reflects new state.
- [ ] AC-4: `ynl --family dpll --do device-set --json '{"id":<id>,"mode":"automatic"}'` flips DPLL mode; DEVICE_CHANGE_NTF received on monitor group.
- [ ] AC-5: Multicast subscribe (`ynl monitor`) on group `monitor` receives PIN_CHANGE_NTF on lock-state flap.
- [ ] AC-6: `dpll_netdev_pin_set` exposes pin via `ethtool --show-tsync` or driver-specific netdev attr.
- [ ] AC-7: `kpf-dpll-selftest` (kselftest skeleton) verifies refcount-balance across register/unregister N times under KASAN.
- [ ] AC-8: Hierarchical pin: parent pin's state change propagates change-ntf for both parent and child.
- [ ] AC-9: rmmod of underlying NIC driver triggers `dpll_device_unregister` + `dpll_pin_unregister`; DEVICE_DELETE_NTF / PIN_DELETE_NTF observed.

## Architecture

`Device` lives in `drivers::dpll::Device`:

```
struct Device {
  id: u32,                              // global xa id
  device_idx: u32,                      // per-driver instance
  clock_id: u64,                        // unique per HW
  type_: DpllType,                      // PPS | EEC | EEC_PPS
  mode_supported: AtomicU64,            // bitmask of dpll_mode
  pin_refs: XArray<u32, Arc<PinRef>>,
  refcount: Refcount,
  registration: Mutex<List<Arc<Registration>>>, // per-driver registrations
}

struct Pin {
  id: u32,
  pin_idx: u32,                         // ida_alloc'd
  prop: Arc<PinProperties>,
  dpll_refs: XArray<u32, Arc<PinRef>>,  // back-refs to dplls
  parent_refs: XArray<u32, Arc<PinRef>>, // for hierarchical pins
  fwnode: Option<Arc<FwnodeHandle>>,
  refcount: Refcount,
  registration: Mutex<List<Arc<Registration>>>,
}

struct PinRef {
  dpll: Arc<Device>,
  pin: Arc<Pin>,
  ops: &'static PinOps,
  priv_: *mut (),
  refcount: Refcount,
}
```

Per-register lifecycle `Device::register`:
1. Acquire `dpll_lock` (global mutex).
2. Look up by (clock_id, dev_driver_id) in `dpll_device_xa`; if exists → reuse, else alloc.
3. Append a `Registration` (carrying ops + priv + module) to `device.registration`.
4. Bump refcount; emit `DPLL_CMD_DEVICE_CREATE_NTF` via notifier chain.

Per-pin register `Pin::register(dpll, pin, ops, priv)`:
1. Validate `pin.prop.type` consistent with `dpll.type` (PPS pin only on PPS-capable DPLLs).
2. Acquire `dpll_lock`.
3. Insert/lookup `PinRef` (dpll, pin) entry in both `dpll.pin_refs` (keyed by pin idx) and `pin.dpll_refs` (keyed by dpll id).
4. Emit `DPLL_CMD_PIN_CREATE_NTF`.

Per-pin-on-pin register `Pin::on_pin_register(parent, child, ops, priv)`:
1. Append `PinRef(parent, child)` to `child.parent_refs`.
2. Per-DPLL that parent is bound to: implicitly bind child via that DPLL with same ops.

Netlink `do/dump` `Netlink::do_device_get`:
1. `genl_split_ops`-dispatch from `dpll_nl.c` policy table.
2. Extract `DPLL_A_ID` (do) or iterate `dpll_device_xa` (dump).
3. For each DPLL: `Device::ops.mode_get(dev, priv, &mode, extack)`, `_lock_status_get`, `_temp_get` (if supported), `_clock_quality_level_get`; `dpll_msg_add_*` packs each attribute.
4. `genlmsg_end`; emit reply.

Netlink `do_pin_set`:
1. Validate `CAP_NET_ADMIN` (`GENL_ADMIN_PERM` in nl_ops flags).
2. Per attribute, call corresponding `dpll_pin_*_set` helper which validates value against `prop.capabilities`/`phase_adjust_min/_max`/`frequency_supported` and invokes `Pin::ops.<setter>` under `dpll_lock`.
3. On success, `__dpll_pin_change_ntf(pin)` emits CHANGE_NTF.

Notifier dispatch `dpll_pin_notify(pin, action)`:
- `raw_notifier_call_chain(&dpll_notifier_chain, action, pin)` — kernel consumers.
- `dpll_pin_event_send(cmd, pin)` — netlink multicast.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

dpll-specific reinforcement:

- **`dpll_lock` global mutex around register/unregister/state-change** — defense against pin-ref race UAF.
- **`pin.prop` immutable once registered** — defense against label-rewrite tricks.
- **`pin_idx_ida` global allocator with `BITS_PER_LONG`-wide pool** — defense against pin-id collision across drivers.
- **`CAP_NET_ADMIN` on every SET operation** — gate enforced by `GENL_ADMIN_PERM` flag in `dpll_nl_ops`.
- **`frequency_supported` allowlist enforced** — `_frequency_set` rejects values not in the supported low/high pairs.
- **`phase_adjust_min/_max` clamp** — `_phase_adjust_set` rejects out-of-range; defense against driver programming illegal phase.
- **netdev-pin clear on netdev unregister** — `unregister_netdevice_notifier`-style cleanup via `dpll_netdev_pin_clear`.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `dpll_device`, `dpll_pin`, `dpll_pin_ref`, `dpll_pin_properties`, and `dpll_pin_registration`; all netlink attribute copies bounded by `nla_strict` policy with `check_object_size`.
- **PAX_KERNEXEC** — `dpll_nl_ops`, `dpll_genl_family`, and per-driver `dpll_device_ops`/`dpll_pin_ops` placed in `__ro_after_init` text; notifier chain head in RO segment.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `dpll_genl_do/_dump` entries and on every `dpll_*_set` setter to defeat stack-spray probing.
- **PAX_REFCOUNT** — saturating refcount on `dpll_device`, `dpll_pin`, and `dpll_pin_ref`; overflow trap defeats register/unregister race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `dpll_device`, `dpll_pin`, `dpll_pin_properties` (including duplicated label strings + freq-range arrays), and `dpll_pin_registration` lists.
- **PAX_UDEREF** — SMAP/PAN enforced on every `do_pin_set`, `do_device_set`, `dpll_msg_add_*` user-pointer copy; reject mis-sized payloads.
- **PAX_RAP / kCFI** — `dpll_device_ops`, `dpll_pin_ops`, and the `dpll_nl_ops[]` dispatch table marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `dpll_device`/`dpll_pin` pointers in `dpll_msg_add_*` debug paths behind CAP_SYSLOG; suppress `%p` in change-ntf tracepoints.
- **GRKERNSEC_DMESG** — restrict DPLL register/unregister and lock-state-flap banners to CAP_SYSLOG so attackers cannot probe network sync topology via dmesg.
- **`DPLL_NL_*` CAP_NET_ADMIN** — every `dpll-*-set` op gated by `GENL_ADMIN_PERM`; default-deny for unprivileged netlink writers.
- **Pin-config NLA strict policy** — `dpll_*_nl_policy` uses `NLA_POLICY_*` exact-type + range; reject malformed nested attributes.
- **Phase-adjust PAX_USERCOPY** — `DPLL_A_PIN_PHASE_ADJUST` user payload copies validated against `prop.phase_adjust_min/_max` before write; defense against signed-overflow producing an illegal phase.
- **Lock-state allowlist** — driver-reported `dpll_lock_status` values validated against the enum; refuse to propagate undefined values that would mislead user-space synchronization daemons.
- **fwnode pinning** — `dpll_pin_fwnode_set` requires the caller to hold a fwnode reference; defense against fwnode-after-free during DPLL teardown.

Rationale: DPLLs steer the timebase for telco-grade NIC ports — a compromised SET operation can desynchronize an entire metro network or force a base station into an out-of-spec frequency. RAP/kCFI on driver ops, refcount-overflow trapping on every device+pin+ref, CAP_NET_ADMIN gating, strict NLA policy on every pin-set attribute, value-range validation on phase-adjust + frequency + lock-state, and SMAP/PAN on every user copy convert the DPLL subsystem from "a register-poking generic-netlink driver" into a structurally hardened sync-plane control surface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-driver bindings (ice/idpf/mlx5/prestera/zl3073x) — each warrants its own Tier-3
- ESYNC pulse format (ITU-T G.8264) — covered if `zl3073x` Tier-3 lands
- linuxptp / ts2phc user-space (out of kernel scope)
- 32-bit DPLL netlink — Rookery is 64-bit only
- Implementation code
