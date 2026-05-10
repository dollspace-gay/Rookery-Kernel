---
title: "Tier-2: drivers/base — device-driver model (bus / class / device / driver core)"
tags: ["tier-2", "drivers", "base", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the Linux Driver Model (LDM) — the foundational abstraction layer underneath every real bus driver (PCI, USB, ACPI, platform, virtio, SPI, I2C, MDIO, auxiliary, …) and every real class driver (block, net, input, sound, power, leds, hwmon, …). LDM owns the bus↔driver match + probe/remove protocol, the `struct device` hierarchy, the kobject + sysfs reflection, the deferred probe machinery, the device-tree / ACPI fwnode unification, the runtime / system PM core, the firmware-loader, the devres allocator, the component aggregator framework, and the devtmpfs filesystem.

Heavily cross-referenced from `fs/sysfs/00-overview.md` (every device shows up under `/sys/devices/...` via the kobject↔kernfs bridge — sysfs is the read-out surface, drivers/base is the producer), `kernel/cgroup/devices.md` (legacy device-cgroup), `arch/x86/cpu-init.md` (CPU sysdev → drivers/base/cpu.c), `drivers/pci/`, `drivers/usb/`, `drivers/platform/`, `drivers/of/` (OF fwnode), `drivers/acpi/` (ACPI fwnode), `kernel/module/` (modprobe alias resolution via `MODULE_DEVICE_TABLE` + `aliases:` sysfs), every future driver Tier-2 wrapper.

Components:

- **device core** (`core.c`): `struct device` lifecycle (`device_register` / `device_add` / `device_create` / `put_device`), kobject-init + sysfs-attribute-group registration, uevent emission, parent/sibling/children list, devres anchor, devtmpfs node creation
- **bus** (`bus.c`): `struct bus_type` registration, per-bus driver list + device list, `match()` callback dispatch, `subsys_register`, sysfs `/sys/bus/<bus>/{drivers,devices}/`
- **driver** (`driver.c`): `struct device_driver` registration, per-driver match-table, sysfs `/sys/bus/<bus>/drivers/<drv>/`
- **class** (`class.c`): `struct class` registration, per-class device list, sysfs `/sys/class/<class>/`, `class_create_file` for class-wide attributes
- **dd** (`dd.c`): bind/unbind state machine — `device_attach`, `driver_attach`, deferred-probe queue (`deferred_probe_pending_list`, `deferred_probe_work`), probe-timeout, async probe
- **devres** (`devres.c`): managed resource allocator — `devm_kmalloc`, `devm_request_irq`, `devm_clk_get`, `devm_*` family; auto-released on driver unbind
- **platform** (`platform.c`): `struct platform_device` + `struct platform_driver`, the bus that holds non-discoverable on-SoC devices (ARM SoC peripherals, x86 ACPI-platform devices, simple-bus DT nodes); also `platform-msi.c` for SoC-style MSI
- **auxiliary** (`auxiliary.c` + `auxiliary_sysfs.c`): the auxiliary bus — used to split a parent device into multiple sub-devices each with its own driver (e.g., mlx5 main + RDMA + Ethernet + DPLL sub-drivers; SOF audio sub-devices)
- **faux** (`faux.c`): faux-device for non-bus root sysdev nodes
- **isa** (`isa.c`): legacy ISA bus shim
- **devtmpfs** (`devtmpfs.c`): kernel-side `/dev` nodefs (CONFIG_DEVTMPFS); auto-creates char/block device nodes when devices register
- **firmware loader** (`firmware.c` + `firmware_loader/`): `request_firmware` / `request_firmware_nowait` / sysfs-fallback loader / firmware sysfs upload (CONFIG_FW_UPLOAD)
- **PM core** (`power/`): runtime PM (`power/runtime.c`), system PM (`power/main.c`), pm_qos (`power/qos.c`), wakeup-source (`power/wakeup.c`), wakeirq (`power/wakeirq.c`), generic_ops, clock_ops; cross-ref future `kernel/power/00-overview.md`
- **fwnode / property** (`property.c` + `swnode.c`): unified abstraction over OF (device-tree) `device_node` + ACPI `fwnode_handle` + software-node; consumers do `device_property_read_*` instead of branching on of_/acpi_
- **regmap** (`regmap/`): mmio + i2c + spi + i3c + sdw + ac97 + mdio register-map abstraction with caching (flat / rbtree / maple) + irq aggregation (`regmap-irq.c`)
- **component framework** (`component.c`): aggregate-driver pattern — N component drivers + 1 master that binds when all N are present (used by drm-bridge, snd-soc cards)
- **transport_class** (`transport_class.c`) + **attribute_container** (`attribute_container.c`): SCSI-style transport-attribute layering
- **container** (`container.c`): hot-pluggable container devices (PCI hotplug aggregator, ACPI memory hotplug)
- **cpu** (`cpu.c`) + **node** (`node.c`) + **memory** (`memory.c`) + **cacheinfo** (`cacheinfo.c`): system-device pseudo-devices for `/sys/devices/system/{cpu,node,memory}/`
- **arch_numa** (`arch_numa.c`) + **arch_topology** (`arch_topology.c`) + **topology** (`topology.c`): generic NUMA + sched-topology surface
- **soc** (`soc.c`): `/sys/bus/soc/` device for SoC identification
- **physical_location** (`physical_location.c`): `physical_location` sysfs attribute group
- **pinctrl** (`pinctrl.c`): pinctrl-binding glue for platform devices
- **module glue** (`module.c`): per-driver module-owner tracking + sysfs symlink to `/sys/module/<name>/drivers`
- **hypervisor** (`hypervisor.c`): `/sys/hypervisor/` topdir
- **syscore** (`syscore.c`): syscore_ops (early-suspend, late-resume) for non-device subsystems
- **devcoredump** (`devcoredump.c`): per-device crash dump pseudo-file
- **trace** (`trace.c` + `trace.h`): tracepoints `dev_pm_qos_*` etc.

### Out of Scope

- Per-Tier-3 (Phase D)
- ARM-only device-tree quirks (`drivers/of/` itself — separate Tier-2 wrapper future)
- ACPI tables parsing (`drivers/acpi/` — separate Tier-2 wrapper future)
- Specific driver implementations
- 32-bit-only paths
- Implementation code

### scope

This Tier-2 governs `/home/doll/linux-src/drivers/base/` (~50 files + `power/` subdir + `firmware_loader/` subdir + `regmap/` subdir) plus the public headers `include/linux/device.h`, `include/linux/device/{bus,class,driver}.h`, `include/linux/{platform_device,property,fwnode,devres,regmap,component}.h`.

### compatibility contract — outline

### sysfs surface

LDM is the **producer** of essentially every `/sys/devices/...` + `/sys/bus/...` + `/sys/class/...` directory. Cross-ref `fs/sysfs/00-overview.md` for the consumer side. Specifically:

- `/sys/bus/<bus>/devices/<dev>` symlinks to `/sys/devices/.../<dev>`
- `/sys/bus/<bus>/drivers/<drv>/{bind,unbind,uevent,module,...}`
- `/sys/bus/<bus>/drivers/<drv>/<dev>` symlink (when bound)
- `/sys/class/<class>/<dev>` symlink to `/sys/devices/.../<dev>`
- `/sys/devices/.../<dev>/{driver,subsystem,uevent,modalias,power/,...}`
- `/sys/devices/system/{cpu,node,memory}/...`

Per-device default attribute set MUST include byte-identical: `uevent` (writable: triggers `add@` event), `modalias` (read: bus-specific match string for udev to invoke `modprobe`), `subsystem` symlink, `driver` symlink (when bound), `power/` group (`autosuspend_delay_ms`, `control`, `runtime_active_time`, `runtime_active_kids`, `runtime_enabled`, `runtime_status`, `runtime_suspended_time`, `runtime_usage`, `async`, `wakeup`, `wakeup_count`, …).

### uevent wire format

Kobject hotplug events broadcast over `NETLINK_KOBJECT_UEVENT` (sequence-numbered, multicast group 1 + per-namespace) to udev / systemd / mdev. Format byte-identical:

```
ACTION@DEVPATH\0
ACTION=add\0
DEVPATH=/devices/.../<dev>\0
SUBSYSTEM=<bus>\0
DEVTYPE=<type>\0
SEQNUM=<n>\0
MODALIAS=<bus>:<bus-specific-encoding>\0
DEVNAME=<n>\0
MAJOR=<n>\0
MINOR=<n>\0
...
```

Cross-ref future `lib/kobject_uevent.md` for the wire-format Tier-3.

### `MODULE_DEVICE_TABLE` / modalias machinery

Each bus driver exports a `<bus>:<glob>` matcher (e.g., `pci:v00008086d*sv*sd*bc*sc*i*`, `usb:v8087p0024*`, `acpi:LNXSYSTM:`, `of:NaTallcPmot1Bjazz`). Userspace `modprobe` resolves to the right module via `/lib/modules/.../modules.alias`. ABI between LDM (which emits `modalias=` in uevent + sysfs) and depmod / modprobe is frozen — keep byte-for-byte identical so distro kmod toolchains keep working.

### Deferred probe

`/sys/kernel/debug/devices_deferred` lists devices currently in deferred-probe state (driver returned `-EPROBE_DEFER` because a dependency wasn't ready). Format byte-identical (one line per deferred device). Probe-timeout configurable via `driver_async_probe=` / `deferred_probe_timeout=` cmdline.

### `bind` / `unbind` sysfs files

`echo <dev> > /sys/bus/<bus>/drivers/<drv>/bind` triggers explicit bind; `echo <dev> > /sys/bus/<bus>/drivers/<drv>/unbind` unbinds. Behavior + permissions identical (root-only via 0200 mode).

### Driver override files

`/sys/devices/.../<dev>/driver_override` writable: forces a specific driver to be considered for this device on next probe pass. Used heavily for VFIO passthrough (`echo vfio-pci > driver_override`). Wire format identical.

### devtmpfs `/dev`

CONFIG_DEVTMPFS auto-mounts a tmpfs-backed `/dev` early in boot (`devtmpfs.mount=1` cmdline, default-on for most distros), and auto-creates `/dev/<name>` for every char/block device that registers via `device_create`. Modes + ownership default to root:root 0600 unless overridden by `devnode()` callback. Behavior identical so existing distros boot without udev rule changes.

### Firmware loader

`request_firmware()` / `request_firmware_nowait()` / `request_firmware_direct()` / `firmware_request_platform()`:
1. Direct fs lookup (`/lib/firmware/<name>`, `/lib/firmware/updates/<name>`, plus per-arch fallbacks).
2. Userspace fallback via `/sys/class/firmware/<name>/{loading,data}` if direct lookup fails AND CONFIG_FW_LOADER_USER_HELPER=y.
3. Cached for hibernate-resume (re-uploaded post-resume).

`/sys/class/firmware/timeout` (write-rw): seconds to wait for userhelper. Wire format + behavior identical.

CONFIG_FW_UPLOAD adds `/sys/class/firmware/<name>/{data,loading,error,remaining_size,status,cancel,timeout}` for userspace push-style firmware updates (FPGA bitstreams, NIC firmware self-update).

### PM core

Runtime PM state machine (`runtime_status`: active, suspended, suspending, resuming) per-device under `/sys/devices/.../power/`. System PM transitions ordered by device tree (parent suspends after children) via `dpm_list`. pm_qos exposed via `/sys/devices/.../power/pm_qos_*`.

Cmdline / sysfs:
- `/sys/power/state` (cross-ref `kernel/power/`) → trigger system suspend/hibernate
- `/sys/power/wakeup_count`
- `/sys/devices/.../power/{control,wakeup,wakeup_count,wakeup_active,...}`

Wire-format identical so userspace `pm-utils` / systemd-logind hibernate works unchanged.

### Component framework

`component_master_add_with_match` + `component_add` / `component_del`. Used by DRM bridges, ASoC sound cards, SoC media-controllers. Deferred-bind until ALL named components register. Behavior identical.

### Auxiliary bus

`/sys/bus/auxiliary/{devices,drivers}` byte-identical layout. `auxiliary_device_init` + `auxiliary_device_add` + `auxiliary_driver_register`. Used by mlx5, intel_idxd, dpll, scmi, etc.

### Regmap

`devm_regmap_init_{mmio,i2c,spi,i3c,mdio,sdw,ac97}` + `regmap_read` / `regmap_write` / `regmap_update_bits` / `regmap_bulk_read` / `regmap_bulk_write`. Backing cache modes selected via `regmap_config.cache_type`: NONE / FLAT / RBTREE / MAPLE. `regmap-irq` aggregates per-chip interrupt status registers into a sub-irq-domain. ABI for in-tree drivers identical (regmap users compile + run unchanged).

### Devres

`devm_kmalloc` / `devm_kzalloc` / `devm_kasprintf` / `devm_kstrdup` / `devm_request_irq` / `devm_clk_get` / `devm_gpiod_get` / `devm_regulator_get` / `devm_phy_get` / etc. — each registers a release callback automatically invoked on driver unbind (post-`->remove()`) or on `device_release()`. Lifetime semantics identical: same call-order, same teardown-order (LIFO). In-tree driver code that uses devm_* compiles + works unchanged.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/base/device-core.md` | `core.c`: `struct device` lifecycle + sysfs/kobject reflection + uevent emission |
| `drivers/base/bus.md` | `bus.c`: `struct bus_type` + `subsys_register` + per-bus iteration |
| `drivers/base/driver.md` | `driver.c`: `struct device_driver` registration + match-table |
| `drivers/base/class.md` | `class.c`: `struct class` + per-class device list |
| `drivers/base/dd.md` | `dd.c`: probe / bind / unbind / deferred-probe state machine + async probe |
| `drivers/base/devres.md` | `devres.c`: managed allocator + LIFO release |
| `drivers/base/platform.md` | `platform.c`: platform bus + `platform_device` + `platform-msi.c` |
| `drivers/base/auxiliary.md` | `auxiliary.c`: auxiliary bus (mlx5, idxd, etc.) |
| `drivers/base/devtmpfs.md` | `devtmpfs.c`: kernel-side `/dev` nodefs |
| `drivers/base/firmware-loader.md` | `firmware.c` + `firmware_loader/`: request_firmware + sysfs-fallback + FW_UPLOAD |
| `drivers/base/component.md` | `component.c`: component-master + component-add framework |
| `drivers/base/property.md` | `property.c` + `swnode.c`: fwnode unification (OF + ACPI + software-node) |
| `drivers/base/regmap-core.md` | `regmap/regmap.c` + `regmap-{flat,rbtree,maple}.c`: regmap core + cache backends |
| `drivers/base/regmap-buses.md` | `regmap-{mmio,i2c,spi,i3c,mdio,sdw,ac97,fsi,sccb}.c`: per-bus regmap shims |
| `drivers/base/regmap-irq.md` | `regmap-irq.c`: regmap-backed irq aggregation |
| `drivers/base/topology.md` | `topology.c` + `arch_topology.c` + `arch_numa.c`: per-cpu/node/memory sysdev surface |
| `drivers/base/cpu.md` | `cpu.c`: per-cpu sysdev (`/sys/devices/system/cpu/`) |
| `drivers/base/node.md` | `node.c`: per-numa-node sysdev |
| `drivers/base/memory.md` | `memory.c`: per-memory-block sysdev (memory hotplug surface) |
| `drivers/base/cacheinfo.md` | `cacheinfo.c`: per-cpu cache topology |
| `drivers/base/devcoredump.md` | `devcoredump.c`: per-device crash dump |
| `drivers/base/transport-class.md` | `transport_class.c` + `attribute_container.c`: SCSI-style transport attribute layering |
| `drivers/base/pm-runtime.md` | `power/runtime.c`: runtime PM state machine |
| `drivers/base/pm-system.md` | `power/main.c`: system suspend/resume ordered traversal |
| `drivers/base/pm-qos.md` | `power/qos.c`: latency / device-state QoS |
| `drivers/base/pm-wakeup.md` | `power/wakeup.c` + `wakeirq.c` + `wakeup_stats.c`: wakeup sources + stats |
| `drivers/base/syscore.md` | `syscore.c`: syscore_ops |

### compatibility outline (top-level)

- REQ-O1: `struct device` ABI for in-tree drivers identical: every documented public field present at same offset is acceptable, but Rookery may reorder private fields if no in-tree code reads them. Macros + accessors (`dev_get_drvdata`, `dev_set_drvdata`, `dev_name`, `dev_to_node`, `dev_dbg`, etc.) byte-equivalent semantics.
- REQ-O2: `/sys/devices/`, `/sys/bus/`, `/sys/class/` topology byte-identical (every device + bus + class present in upstream baseline appears at the same path with the same default attribute set).
- REQ-O3: uevent netlink wire format byte-identical (udev / systemd / mdev consume unchanged).
- REQ-O4: `MODULE_DEVICE_TABLE` / modalias mechanism byte-identical (depmod / modprobe / kmod consume unchanged).
- REQ-O5: Deferred-probe semantics + `/sys/kernel/debug/devices_deferred` format identical.
- REQ-O6: `bind` / `unbind` / `driver_override` sysfs files behave identically (VFIO + manual driver-rebind workflows work unchanged).
- REQ-O7: devtmpfs `/dev` auto-creates char/block device nodes identically (CONFIG_DEVTMPFS=y).
- REQ-O8: `request_firmware()` direct + sysfs-fallback path byte-identical; `/sys/class/firmware/timeout` writable.
- REQ-O9: PM core `/sys/devices/.../power/` attributes byte-identical; runtime/system PM ordering preserved.
- REQ-O10: Component framework + auxiliary bus + regmap APIs identical for in-tree consumers (recompile-clean).
- REQ-O11: Devres `devm_*` family identical lifetime semantics (LIFO release on unbind).
- REQ-O12: TLA+ models declared at this Tier-2 (probe state machine, deferred-probe queue, device-tree teardown ordering).
- REQ-O13: Verus/Creusot Layer-4 functional contracts on devres release-ordering + kref refcount integrity.
- REQ-O14: Hardening: row-1 features applied per `00-security-principles.md`, with LDM-specific reinforcement (privileged sysfs writes audit-logged via LSM hook, driver_override change requires CAP_SYS_ADMIN).

### acceptance criteria (top-level)

- [ ] AC-O1: A reference Debian image boots to multi-user.target on Rookery — every device under `/sys/devices/` enumerates with the same paths + same default attribute set as upstream baseline. (covers REQ-O1, REQ-O2)
- [ ] AC-O2: `udevadm monitor --kernel` shows byte-identical uevent stream during boot vs upstream. (covers REQ-O3)
- [ ] AC-O3: `cat /lib/modules/$(uname -r)/modules.alias | head` and `find /sys/devices -name modalias -exec cat {} \;` produce byte-identical content vs upstream baseline. (covers REQ-O4)
- [ ] AC-O4: `cat /sys/kernel/debug/devices_deferred` mid-boot shows the expected deferred set; deferred probes complete identically. (covers REQ-O5)
- [ ] AC-O5: VFIO `driver_override` workflow: `echo vfio-pci > /sys/bus/pci/devices/0000:01:00.0/driver_override; echo 0000:01:00.0 > /sys/bus/pci/drivers/<orig>/unbind; echo 0000:01:00.0 > /sys/bus/pci/drivers/vfio-pci/bind` succeeds and rebinds. (covers REQ-O6)
- [ ] AC-O6: `mount -t devtmpfs none /tmp/dev` works; entries auto-appear when devices register. (covers REQ-O7)
- [ ] AC-O7: A driver that uses `request_firmware("foo.bin")` retrieves the file from `/lib/firmware/foo.bin`; sysfs-fallback path triggers when file missing AND CONFIG_FW_LOADER_USER_HELPER=y. (covers REQ-O8)
- [ ] AC-O8: `systemctl suspend` followed by lid-open resumes correctly: every device's `runtime_status` returns to `active`. (covers REQ-O9)
- [ ] AC-O9: mlx5 (auxiliary-bus consumer) loads + binds correctly with all sub-devices. (covers REQ-O10)
- [ ] AC-O10: A driver that `devm_kmalloc`s in probe + does not free in remove gets the memory released on unbind. KASAN-clean test. (covers REQ-O11)
- [ ] AC-O11: kselftest `tools/testing/selftests/firmware/` passes. (covers REQ-O8)
- [ ] AC-O12: kselftest `tools/testing/selftests/devices/` passes (probe enumeration sanity). (covers REQ-O1, REQ-O2)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/drivers-base/probe_state.tla` | `drivers/base/dd.md` (proves: probe state machine — UNBOUND → MATCHED → BINDING → BOUND → UNBINDING → UNBOUND; concurrent bind/unbind/driver_override never produce double-probe or use-after-free of `device_driver`) |
| `models/drivers-base/deferred_probe.tla` | `drivers/base/dd.md` (proves: deferred-probe queue eventually drains under any partial ordering of dependency satisfaction; no livelock; probe-timeout fires after `deferred_probe_timeout` seconds) |
| `models/drivers-base/dpm_ordering.tla` | `drivers/base/pm-system.md` (proves: system suspend / resume ordering on `dpm_list` respects parent↔child + supplier↔consumer + class-suspend-late constraints; cycle detection via fwnode_link prevents deadlock) |
| `models/drivers-base/devres_release.tla` | `drivers/base/devres.md` (proves: devres release-list LIFO-ordered; concurrent `devm_remove_action` + driver-unbind never double-free or skip a release callback) |
| `models/drivers-base/uevent_seqnum.tla` | `drivers/base/device-core.md` (proves: uevent SEQNUM monotonic across all CPUs; multicast send + per-namespace queueing preserves total order udev sees) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/base/device-core.md` | `device_register` post-condition: kobject refcount == 1, sysfs-attribute-group fully created or fully rolled back, no partial state |
| `drivers/base/devres.md` | `devm_kmalloc` invariant: every allocation has a matching release node in dev->devres_head; release-on-unbind frees every node |
| `drivers/base/dd.md` | `device_attach` invariant: at most one driver simultaneously bound to a device; bind transitions are atomic w.r.t. driver_override write |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-device `kobject` refcount + per-driver refcount + per-class refcount + per-bus refcount use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | every `struct bus_type`, `struct device_driver`, `struct class`, `struct attribute_group`, `dev_pm_ops` `static const` (these are compile-time constants in upstream — preserve in Rookery) | § Mandatory |
| **SIZE_OVERFLOW** | devres slab arithmetic + sysfs-attribute count multiplications use checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | bus / class / driver vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed `struct device` memory zeroed (carries `dev->driver_data` private state per-driver) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | LDM has no JIT — N/A surface-side, but enforced for any consumer | § Mandatory |

### Row-2 (LSM-stackable) features for LDM

LSM hooks called: `security_dev_*` not currently a top-level family — driver-bind events go through file LSM hooks on the sysfs `bind` / `unbind` / `driver_override` files. SELinux + AppArmor + future GR-RBAC stackable LSM each mediate via the sysfs MAC hooks.

GR-RBAC adds:
- Per-role disallow of `driver_override` write (prevents container-escape via VFIO rebind).
- Per-role disallow of bind/unbind on specific buses (PCI, USB).
- Per-role audit of `request_firmware` calls + which firmware path was used.

### LDM-specific reinforcement

- **driver_override write requires CAP_SYS_ADMIN** in Rookery (upstream allows root-write only via 0644-mode sysfs file, which is functionally equivalent on a single-user system, but explicit cap-check is more portable and audit-friendly).
- **Firmware loader path-traversal guard**: `request_firmware("../../etc/shadow")` MUST be rejected. (Upstream rejects via `name_to_dev_t` quirks; Rookery enforces via canonical-path validation.)
- **Devtmpfs node-creation**: never traverse symlinks; dirent-by-dirent creation only.
- **Auxiliary / component / regmap consumer-pointer validation**: every callback fetched via `container_of` MUST pre-validate the `device_type` field (defense-in-depth against type confusion if a buggy driver registers under the wrong parent type).
- **Deferred-probe denial-of-service guard**: cap deferred-probe-list length per-bus to 1024 (upstream is unbounded; a malicious USB device claiming many endpoints could exhaust kernel memory). Configurable via `deferred_probe_max=` cmdline.

