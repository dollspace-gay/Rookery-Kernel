---
title: "Tier-3: drivers/base/core.c — `struct device` lifecycle + sysfs/kobject reflection + uevent emission"
tags: ["tier-3", "drivers-base", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`struct device` is the universal node in the LDM (Linux Driver Model) tree — every PCI / USB / I2C / SPI / platform / virtio / SoC peripheral / class-device / sysdev appears as one. This Tier-3 covers `drivers/base/core.c` (~5300 lines) which implements the per-device lifecycle: `device_initialize` → `device_add` → `device_register` → `put_device` → `device_release`. Owns:

- per-device kobject embedding + sysfs-attribute-group registration
- per-device parent / sibling / child list mgmt + `class` membership + `bus` membership
- uevent (NETLINK_KOBJECT_UEVENT) emission with action / DEVPATH / SUBSYSTEM / DEVTYPE / SEQNUM / per-bus + per-class env vars
- devres anchor (`device_release` invokes devres release-list LIFO)
- devtmpfs node creation (`device_create_file` + `device_create_with_groups` triggering devtmpfs work)
- per-device fwnode handle attachment (OF / ACPI / software-node)
- per-device-link supplier↔consumer dependency graph (used by deferred-probe + suspend/resume ordering)
- per-device runtime PM init + accounting (cross-ref `drivers/base/pm-runtime.md`)
- per-device DMA API setup (`device_set_dma_ops`, `dma_coerce_mask_and_coherent`)

### Acceptance Criteria

- [ ] AC-1: A reference Debian image boots; every device under `/sys/devices/` has the same sysfs path + default attribute set as upstream baseline. (covers REQ-1)
- [ ] AC-2: `udevadm monitor --kernel` shows byte-identical uevent stream during boot vs upstream. (covers REQ-2)
- [ ] AC-3: SEQNUM reads strictly increasing per-uevent in stress test (1000-fold add+remove of vfio-bound device). (covers REQ-3)
- [ ] AC-4: Refcount stress test — concurrent `device_add` + `device_unregister` + `get_device`/`put_device` from N tasks; final refcount==0 + no UAF (KASAN clean). (covers REQ-4)
- [ ] AC-5: devres release-LIFO test — driver does `devm_kmalloc(A)` + `devm_kmalloc(B)` in probe; release order on unbind is B → A. (covers REQ-5)
- [ ] AC-6: Device-link test — consumer driver bind blocks while supplier driver is unbound; auto-bind upon supplier-bind; unbind sequence: consumer first, then supplier. (covers REQ-6)
- [ ] AC-7: Per-class uevent test — class with `dev_uevent` callback (e.g., input class) injects extra `KEY=`, `EV=` env vars. (covers REQ-7)

### Architecture

`Device` struct ownership lives in `kernel::device::Device` — a `KBox<DeviceInner>` with embedded `kobject::KObject`, parent/child `Mutex<Vec<Arc<Device>>>`, per-bus / per-class `Option<&'static dyn BusType>` / `Option<&'static dyn Class>`, devres anchor `Mutex<DevresList>`, fwnode `AtomicPtr<FwnodeHandle>`, supplier/consumer `Mutex<Vec<DeviceLink>>`, runtime PM `RtpmState` (cross-ref `drivers/base/pm-runtime.md`), DMA ops `AtomicPtr<dyn DmaMapOps>` (cross-ref `kernel/dma/00-overview.md`).

`Device::register()`:
1. `kobject_init` (refcount=1, ktype=device_ktype).
2. Apply per-class `dev_uevent_filter`.
3. `kobject_add` (creates `/sys/devices/.../<name>` dir).
4. Create device-default attribute group (`uevent`, `power/`, `modalias`, `subsystem`/`driver` symlinks).
5. `bus_add_device` (links to `/sys/bus/<bus>/devices/<dev>`).
6. `class_dev_iter_init` for class-add.
7. `device_create_sys_dev_entry` (devtmpfs node create when `dev != NULL`).
8. `kobject_uevent(KOBJ_ADD)` (emits to NETLINK_KOBJECT_UEVENT + invokes per-namespace uevent_helper if set).

`Device::release()` (called when refcount hits 0):
1. devres release-list drained LIFO.
2. Per-bus `dev_release` invoked.
3. Per-class `dev_release` invoked.
4. Per-dev_type `dev_release` invoked.
5. Final `kfree`.

uevent emission via `kobject_uevent_env(kobj, action, envp)`:
- per-uevent struct kobj_uevent_env populated: ACTION + DEVPATH + SUBSYSTEM + per-bus envp (via `bus_uevent`) + per-class envp (via `class_uevent`) + per-dev_type envp (via `dev_uevent_filter`).
- SEQNUM allocated atomically (`uevent_seqnum.fetch_add(1, Relaxed)` — but with global ordering preserved via per-cpu serialization).
- Multicast on `NETLINK_KOBJECT_UEVENT` group 1 (per-namespace netns_id).
- If `uevent_helper` cmdline set, exec'd as a subprocess (legacy `/sbin/hotplug` mechanism — Rookery preserves but distros set empty).

### Out of Scope

- per-bus details (each `bus_type` has its own Tier-3 — `drivers/base/bus.md`)
- per-class details (`drivers/base/class.md`)
- devres internals (`drivers/base/devres.md`)
- runtime PM internals (`drivers/base/pm-runtime.md`)
- fwnode internals (`drivers/base/property.md`)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct device` | per-device root state | `kernel::device::Device` |
| `device_initialize(dev)` | zero-init kobject + lists + locks; refcount=1 | `Device::new` |
| `device_add(dev)` | full registration: kobject_add + sysfs + bus_add + class_add + uevent | `Device::add` |
| `device_register(dev)` | `device_initialize` + `device_add` combined | `Device::register` |
| `device_create(class, parent, devt, drvdata, fmt, ...)` | class device + chardev region | `Device::create_in_class` |
| `device_create_with_groups(...)` | + attribute groups | `Device::create_with_groups` |
| `device_destroy(class, devt)` | class device removal | `Device::destroy_in_class` |
| `device_unregister(dev)` | inverse of `device_register` | `Device::unregister` |
| `device_del(dev)` | inverse of `device_add` (no put) | `Device::del` |
| `put_device(dev)` | refcount decrement → release on 0 | `Drop` impl on `Arc<Device>` |
| `get_device(dev)` | refcount increment | `Arc::clone` |
| `device_for_each_child[_reverse](...)` | iterate child devices | `Device::iter_children` |
| `device_find_child[_by_name](...)` | find child by predicate | `Device::find_child*` |
| `device_link_add(consumer, supplier, flags)` | supplier↔consumer dep | `Device::link_to` |
| `device_link_remove(consumer, supplier)` | remove link | `Device::unlink_from` |
| `dev_set_name(dev, fmt, ...)` | set kobject name | `Device::set_name` |
| `dev_set_drvdata(dev, p)` / `dev_get_drvdata(dev)` | per-driver private | `Device::driver_data*` |
| `device_create_file(dev, attr)` / `_remove_file` | per-attribute file | `Device::create_attribute` |
| `sysfs_create_groups(&dev->kobj, groups)` | create attribute group set | `Device::create_groups` |
| `kobject_uevent(&dev->kobj, action)` | emit uevent | `Device::emit_uevent` |
| `kobject_uevent_env(...)` | emit uevent with custom env vars | `Device::emit_uevent_with_env` |
| `device_change_owner(dev, kuid, kgid)` | userns ownership change | `Device::change_owner` |
| `device_iommu_mapped(dev)` | IOMMU presence test | `Device::has_iommu` |
| `device_get_match_data(dev)` | per-driver match data from of/acpi | `Device::match_data` |
| `device_property_*` family | fwnode property accessors | (see `drivers/base/property.md`) |

### compatibility contract

REQ-1: Per-device sysfs path (`/sys/devices/<bus-path>/<dev>`) byte-identical to upstream baseline for every in-tree device — same parent traversal, same naming convention, same default attribute set (`uevent`, `modalias`, `subsystem`, `driver` symlink (when bound), `power/`, per-bus `devtype`).

REQ-2: uevent over NETLINK_KOBJECT_UEVENT byte-identical wire format including the `\0`-delimited env-var encoding + per-bus + per-class extension (e.g., USB adds `BUSNUM=`, `DEVNUM=`, `DEVNAME=`, `PRODUCT=`, `TYPE=`).

REQ-3: SEQNUM monotonic across all CPUs (uevent ordering invariant — udev relies on this).

REQ-4: Per-device refcount integrity — `device_register` returns a device with refcount==1; every `get_device` matched by exactly one `put_device` until release.

REQ-5: `device_release` invokes devres release-list LIFO before kobject release; per-driver-data pointer is freed by per-driver `release` callback (driver responsibility, not core).

REQ-6: Device-link supplier↔consumer ordering preserved across runtime PM + system PM + deferred-probe (consumer cannot resume before supplier; consumer cannot probe before supplier is bound).

REQ-7: per-bus `bus_type::add_dev` / `class::dev_uevent` / `dev_type::release` callbacks invoked in correct order at the documented point in the lifecycle.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dev_no_use_after_release` | UAF | Every `Arc<Device>` outliving its refcount is impossible (Rust borrow check + `Arc` invariants — Kani-verified for the Rust wrapper). |
| `seqnum_no_overflow` | OVERFLOW | uevent_seqnum AtomicU64 + saturate-on-overflow (defense against 2^64-event-flood; Kani proves saturation correctness). |

### Layer 2: TLA+

`models/drivers-base/uevent_seqnum.tla` (declared in parent Tier-2): proves SEQNUM monotonicity across all CPUs + per-namespace queueing preserves total order udev sees.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `device_register` post-condition: kobject refcount == 1, attribute group fully created OR fully rolled back, no partial state | `Device::register` |
| `device_link_add` post-condition: link added to BOTH supplier consumers list AND consumer suppliers list atomically | `Device::link_to` |

### Layer 4: Verus/Creusot functional

`Device::register` ↔ `device_register` (upstream) round-trip: a device registered via Rookery is consumed by udev/systemd-journald with byte-identical uevent stream. Encoded as a Verus model: `forall dev. uevent_envp(dev) == upstream_uevent_envp(dev)`.

### hardening

(Inherits row-1 features from `drivers/base/00-overview.md` § Hardening.)

LDM-specific reinforcement here:

- **Per-device kobject refcount uses `Refcount` (saturating)** — overflow saturates instead of wraps; defense against ref-count-overflow CVE class.
- **uevent envp size cap** — per-uevent envp <= 32KB total + <= 64 entries (defense against malformed-driver flooding userspace udev with huge env vars).
- **SEQNUM monotonicity** preserved across uevent_helper exec window (uevent_seqnum increment happens BEFORE userspace fork; never re-incremented mid-flight).
- **Device-link cycle detection** at link_add time — refuse cycles (defense against driver bug causing supplier↔consumer cycle which would deadlock dpm_list traversal).

