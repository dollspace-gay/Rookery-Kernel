# Tier-3: drivers/accel/ — Compute Accelerator Subsystem (drm_accel core + per-vendor accel minors)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/accel/drm_accel.c
  - include/drm/drm_accel.h
  - drivers/accel/Kconfig
  - drivers/accel/Makefile
-->

## Summary

The `drivers/accel/` subsystem hosts Linux's compute-only accelerator drivers — a DRM-derived branch that reuses the `drm_device` core (`gem_objects`, `dma_buf`, `drm_file`, `drm_ioctl` plumbing, drm-managed resources) while exposing the device through a separate `accel/accelN` character-device minor space carved out of major number 261 (`ACCEL_MAJOR`) instead of the DRM major (226). Drivers set the `DRIVER_COMPUTE_ACCEL` feature bit on `drm_driver`, and the accel core (`drivers/accel/drm_accel.c`) provides the `accel_class` sysfs class, the `accel_devnode` factory (yielding `/dev/accel/accelN`), the `accel_minors_xa` allocator, the `accel_open()` file-operations entry point (re-used by every accel driver via `DEFINE_DRM_ACCEL_FOPS`), and the debugfs/sysfs glue under `/sys/class/accel/`. By splitting the character-device namespace, the accel branch lets non-graphics compute hardware (Habana Gaudi, Intel NPU, Qualcomm Cloud AI 100, AMD XDNA, Rockchip Rocket NPU, ARM Ethos-U) participate in the DRM/GEM ecosystem without colliding with the graphics ABI or appearing as DRI devices.

This overview Tier-3 covers `drm_accel.c` (~208 lines: sysfs class, minor allocation, stub fops, major registration) plus the cross-cutting policy that binds the per-vendor Tier-3 docs (`habanalabs.md`, `qaic.md`, future `ivpu.md`, `amdxdna.md`, `rocket.md`, `ethosu.md`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `ACCEL_MAJOR` (= 261) | char-major reserved for accel minors | `drivers::accel::MAJOR` |
| `accel_minors_xa` | XArray-backed minor-number allocator (parallel to `drm_minors_xa`) | `drivers::accel::MinorAlloc` |
| `accel_class` | `/sys/class/accel/` device class | `drivers::accel::SysfsClass` |
| `accel_devnode(dev, mode)` | builds `accel/accelN` devnode names | `drivers::accel::devnode` |
| `accel_set_device_instance_params(kdev, idx)` | `MKDEV(ACCEL_MAJOR, idx)` + class/type wiring | `drivers::accel::set_instance_params` |
| `accel_open(inode, filp)` | per-fd open: acquire minor, replace fops, invoke driver `open()` | `drivers::accel::open_file` |
| `accel_stub_open(inode, filp)` | first-touch fops replacement (called from char-major fops) | `drivers::accel::stub_open` |
| `accel_debugfs_register(dev)` | per-device debugfs root under `dev->debugfs_root` | `drivers::accel::debugfs::register_dev` |
| `accel_core_init()` / `_exit()` | `register_chrdev(ACCEL_MAJOR, ...)` + sysfs class | `drivers::accel::Subsystem::init` / `_exit` |
| `DEFINE_DRM_ACCEL_FOPS(name)` | macro: per-driver `file_operations` with `accel_open` + `drm_release` + `drm_ioctl` | `drivers::accel::AccelFops` |
| `DRIVER_COMPUTE_ACCEL` (`drm_driver.driver_features`) | feature bit marking a driver as accel | `drivers::drm::DriverFeatures::COMPUTE_ACCEL` |

## Compatibility contract

REQ-1: `ACCEL_MAJOR == 261` is fixed in `include/uapi/linux/major.h`-equivalent (here `include/drm/drm_accel.h`); userland depends on it for `mknod` / udev rules; Rookery preserves the literal value.

REQ-2: Per-device minor allocated from the same XArray (`accel_minors_xa`) so simultaneously-loaded accel drivers cannot collide minors.

REQ-3: `/dev/accel/accelN` devnode path produced by `accel_devnode()`; udev `accel` rules (and `libdrm`'s accel-aware path) depend on the exact `accel/accel%d` template.

REQ-4: `/sys/class/accel/` populated with one device per registered accel-feature `drm_device`; `device_type == accel_sysfs_device_minor` distinguishes from primary/control DRM minors.

REQ-5: `accel_open()` shares `f_mapping` across all char-devs of a single device (re-uses `dev->anon_inode->i_mapping`) — `drm_open_helper` semantics preserved.

REQ-6: When `DRIVER_COMPUTE_ACCEL` is set, the DRM core takes the accel-minor path (`drm_dev_register` allocates from `accel_minors_xa`, not `drm_minors_xa`); render and primary minors are not allocated.

REQ-7: Default debugfs entry `accel/<dev>/name` lists driver name + device + master/unique; tracepoints (`drm_minor_acquire/release`) shared with DRM.

REQ-8: Accel drivers MUST NOT register graphics ioctls; DRM core filters per-`DRM_IOCTL_DEF_DRV` flags to reject DRM_AUTH/DRM_MASTER paths that don't apply to compute.

REQ-9: `dma-buf` import/export shared with DRM — accel devices participate in cross-driver buffer sharing (e.g., NIC → accelerator zero-copy).

REQ-10: Per-driver `module_init` ordering: accel-core (`subsys_initcall`-like via DRM core init) before any accel driver `module_init`.

## Acceptance Criteria

- [ ] AC-1: `cat /proc/devices | grep accel` shows `261 accel`.
- [ ] AC-2: With one accel driver loaded, `ls /dev/accel/` shows `accel0`; `udevadm info /dev/accel/accel0` shows `SUBSYSTEM=accel`.
- [ ] AC-3: `ls /sys/class/accel/` shows the per-device entry with valid `device` symlink to PCI/platform parent.
- [ ] AC-4: With two accel drivers loaded simultaneously (Habana + ivpu), each gets unique minor (`accel0`, `accel1`).
- [ ] AC-5: Userspace can open `/dev/accel/accelN`, call `DRM_IOCTL_VERSION` and receive driver name + DRIVER_COMPUTE_ACCEL.
- [ ] AC-6: `dma-buf` exported from accel device imports into a DRM render node and vice-versa.
- [ ] AC-7: Debugfs `accel/accel0/name` returns one line matching driver `name=`.

## Architecture

The accel core is a thin shim layered on `drm_minor`:

```
ACCEL_MAJOR=261 chrdev (accel_stub_fops)
       │
       ▼  iminor(inode) → minor lookup in accel_minors_xa
accel_stub_open  → replace_fops(filp, drv->fops)
       │
       ▼
DEFINE_DRM_ACCEL_FOPS-generated per-driver fops
       │
       ▼
accel_open  → drm_minor_acquire(accel_minors_xa) → drm_open_helper
       │
       ▼
drm_driver.open(struct drm_device *, struct drm_file *)
```

`drm_dev_register()` (in DRM core) sees `DRIVER_COMPUTE_ACCEL` and:
1. Allocates a minor via `drm_minor_alloc(dev, DRM_MINOR_ACCEL)` which uses `accel_minors_xa`.
2. Calls `accel_set_device_instance_params(minor->kdev, minor->index)` → sets `devt = MKDEV(261, idx)`, `class = &accel_class`, `type = &accel_sysfs_device_minor`.
3. `device_add(minor->kdev)` creates `/sys/class/accel/accelN` and `/dev/accel/accelN`.

`accel_debugfs_register` builds the `accel/accelN/` debugfs directory (re-using `dev->debugfs_root` to avoid duplicating the drm/ root for compute devices).

Per-driver lifecycle:
- `module_init`: `drm_dev_alloc(&driver, parent)` → set `driver_features = DRIVER_COMPUTE_ACCEL | …` → `drm_dev_register` → minor allocated, sysfs/devnode created.
- `open()`: dev-fd opens; `drm_file` allocated, driver `open()` callback fires (per-fd `hl_fpriv` / `qaic_user` / `ivpu_file_priv` …).
- `ioctl()`: drm_ioctl dispatches via per-driver `drm_ioctl_desc[]` table; `DRM_IOCTL_DEF_DRV(VENDOR_OP, fn, flags)`.
- `mmap()`: per-driver `mmap` op (e.g., command-buffer or BO mapping).
- `release()`: `drm_release` + driver `postclose`; per-fd context freed.

Drivers in tree (each gets its own Tier-3 over time):
- `habanalabs/` — Habana/Intel Gaudi/Goya AI accelerator (this overview's sibling `habanalabs.md`).
- `qaic/` — Qualcomm Cloud AI 100 (sibling `qaic.md`).
- `ivpu/` — Intel Versatile Processing Unit (Meteor Lake / Lunar Lake NPU).
- `amdxdna/` — AMD XDNA (Ryzen AI / Phoenix NPU).
- `rocket/` — Rockchip RK3588 NPU.
- `ethosu/` — ARM Ethos-U microNPU.

## Hardening

- **Per-driver `DRIVER_COMPUTE_ACCEL` enforced** — DRM core refuses to allocate accel minor if the bit is missing.
- **Accel minors decoupled from DRM minors** — graphics-stack bugs cannot exhaust the accel minor space and vice-versa.
- **`accel_stub_fops` indirection** — first open() goes through stub which then `replace_fops` to driver fops; ensures driver cannot bypass the minor-lookup path.
- **`xa_empty` WARN on exit** — module unload ordering bug (driver unloads before accel-core) trips a WARN.
- **`atomic_fetch_inc(open_count)` balanced with `atomic_dec` on error path** — refcount cannot underflow on open failure.
- **`fops_get` failure → `-ENODEV`** — racing driver unload during open returns cleanly.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `drm_file`, per-driver `_fpriv` (habana `hl_fpriv`, qaic `qaic_user`), and accel-debugfs scratch buffers; reject userspace copies that span allocation boundaries.
- **PAX_KERNEXEC** — `drm_accel.c` text in W^X kernel image; `accel_class`, `accel_devnode`, `accel_stub_fops`, and `accel_sysfs_device_minor` placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `accel_open`, `accel_stub_open`, and per-driver `drm_ioctl` entry to defeat stack-layout-leak primitives.
- **PAX_REFCOUNT** — saturating `refcount_t` on `drm_device`, `drm_file`, `drm_minor` so per-fd / per-minor reference accounting cannot overflow into UAF.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `drm_file`, per-driver context, BO metadata, and debugfs scratch so stale device-handle / DMA-mapping data cannot bleed into a re-issued fd.
- **PAX_UDEREF** — SMAP/PAN enforced on every accel ioctl entry; per-driver UAPI structs copied via `copy_from_user`/`copy_to_user` only, never raw dereferenced.
- **PAX_RAP / kCFI** — `drm_driver.ioctls`, `drm_driver.fops`, `drm_driver.open/postclose`, `accel_stub_fops`, and `accel_class.devnode` marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of accel-driver symbols behind CAP_SYSLOG; suppress `%p` of `drm_device`/`drm_file` pointers in accel tracepoints (`drm_minor_acquire/release`).
- **GRKERNSEC_DMESG** — restrict accel-init banners, per-device probe messages, and driver-fault reports to CAP_SYSLOG so probing tools cannot enumerate accelerator presence via dmesg.
- **Accel-cdev permission gates** — `/dev/accel/accelN` owned `root:render` (or per-distro `accel` group) with 0660; udev rules MUST NOT widen perms to world-readable.
- **DRM-derived ioctl filter** — every accel-driver ioctl table audited for missing `DRM_RENDER_ALLOW` and presence of DRM_MASTER/DRM_AUTH flags that are nonsensical for compute; reject mis-flagged tables at register time.
- **dma-buf isolation** — `drm_gem_prime_import/export` allowlist-checked per accel driver; refuse cross-namespace dma-buf transfer that crosses a user-ns boundary unless explicit CAP_SYS_ADMIN in the owning ns.
- **Debugfs entries CAP_SYS_RAWIO + 0400** — `accel/accelN/name` and per-driver debugfs files require CAP_SYS_RAWIO; suppress device-address disclosure.
- **GRKERNSEC_HIDESYM in trace** — `trace_drm_minor_acquire`/`_release` event fields scrub `drm_device *`, `drm_file *`, and per-driver context pointers (replace `%p` with `%pK`-equivalent hashing).

Rationale: accelerator devices are full DMA bus-masters with privileged access to host memory, often loaded with vendor firmware whose attack surface (CS submit, BO map, manage-ioctl, MHI channels, SAHARA boot) was historically GPU-driver-grade fragile. Pinning the accel-cdev minor namespace, refcount-trapping per-fd state, filtering DRM ioctl flags, allowlisting dma-buf transfers, and hiding kallsyms/dmesg trace data turn `/dev/accel/` from "graphics-stack-shaped attack surface" into a structurally narrower compute-only boundary.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `minor_alloc_no_collision` | UNIQUENESS | `accel_minors_xa` allocator never re-issues an active minor index. |
| `accel_open_refcount_balanced` | UAF | per-open `atomic_fetch_inc(open_count)` paired with `atomic_dec` on every error path. |
| `stub_open_replace_fops_ordered` | ORDERING | `replace_fops` runs before any driver-fops dispatch; concurrent unload safe via `fops_get`. |

### Layer 2: Verus / Creusot

- Open-fd flow invariant: `accel_open` returns success ⇒ `drm_file` allocated and registered in `drm_device.filelist` with `drm_file.minor` pointing to the acquired accel minor; on error every reference released.
- Minor lifecycle: `drm_minor_acquire` and `drm_minor_release` bracket every fd; XArray empty at `accel_core_exit`.

### Layer 3: Lock-discipline

`drm_device.master_mutex` held for the duration of `accel_name_info`'s `seq_printf`; refcounted `drm_master *` is the only pointer dereferenced under the lock.

## Open Questions

(none at this overview tier)

## Out of Scope

- DRM graphics ioctls / KMS / GEM-graphics (covered in `drivers/gpu/drm/` Tier-3s)
- Per-vendor command-submission internals (covered in `habanalabs.md`, `qaic.md`, future `ivpu.md`, `amdxdna.md`, `rocket.md`)
- Firmware-loading internals (per-driver Tier-3s)
- Implementation code
