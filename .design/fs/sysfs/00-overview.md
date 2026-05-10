# Tier-2: fs/sysfs — sysfs (kobject-based device-driver model)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/sysfs/
  - fs/kernfs/
  - lib/kobject.c
  - lib/kobject_uevent.c
  - include/linux/kobject.h
  - include/linux/sysfs.h
  - include/linux/kernfs.h
-->

## Summary
Tier-2 overview for sysfs — Linux's kobject-based pseudo-filesystem exposing the kernel device-driver model + every device/driver/bus/class as a `/sys/*` directory tree. Built on top of the `kernfs` lower-layer filesystem (which also backs cgroupfs + configfs). Used by:
- `udev` / `udevd` for device-event handling (consumes uevents from `/sys/*/uevent` files)
- `systemd` for socket-activation, device-activation, target-units (`systemd-devmgr`)
- `lsblk`, `lsusb`, `lspci`, `dmidecode` (parses /sys for device topology)
- `ethtool`, `iw`, `nmcli` (per-device config exposed via /sys/class/net + /sys/class/ieee80211)
- Per-cgroup file API (cgroup-v1 used sysfs-style; cgroup-v2 uses cgroupfs which shares kernfs backbone)

The tree exposes:
- `/sys/devices/` — device topology (per-bus per-device hierarchy)
- `/sys/bus/` — per-bus device + driver lists
- `/sys/class/` — per-class device groupings (block / net / tty / hidraw / etc.)
- `/sys/block/` — block devices (deprecated symlinks to /sys/class/block)
- `/sys/firmware/` — firmware info (ACPI / EFI / DMI)
- `/sys/fs/` — per-filesystem state (cgroup / btrfs / etc.)
- `/sys/kernel/` — kernel state (debug / live-patch / mm / config / notes / security)
- `/sys/module/` — per-loaded-module state
- `/sys/power/` — system suspend/resume control
- `/sys/dev/` — major:minor symlinks
- `/sys/hypervisor/` — hypervisor info (Xen / Hyper-V)

Three core abstractions in `lib/kobject.c`:
- `struct kobject` — per-instance object with refcount + parent + name
- `struct kobj_type` (ktype) — describes attributes + show/store ops
- `struct kset` — set/container of kobjects (per-bus / per-class)

Plus the `uevent` infrastructure (`kobject_uevent.c`) emits `add`/`remove`/`change`/`bind`/`unbind` events to userspace netlink consumers (udev).

Sub-tier-2 of `fs/00-overview.md`. Sibling of `fs/proc/00-overview.md` (now complete).

## Scope

This Tier-2 governs all of `/home/doll/linux-src/fs/sysfs/` (~6 source files), the `kernfs` lower-layer (`fs/kernfs/`), the kobject framework (`lib/kobject.c` + `lib/kobject_uevent.c`), plus the public API headers. Per-file Tier-3 docs live under `.design/fs/sysfs/`.

## Compatibility contract — outline

### Mount semantics

`mount -t sysfs sysfs /sys` (default mount per init scripts). No mount options for the most part.

### Top-level subdirectory layout

```
/sys/
├── block/      (legacy alias for /sys/class/block)
├── bus/
├── class/
├── dev/
├── devices/    (master device topology)
├── firmware/
├── fs/
├── hypervisor/
├── kernel/
├── module/
└── power/
```

Identical layout (each parent kobject registered at boot).

### Attribute file convention

Each sysfs file represents a single value (string / int / boolean) with read returning current value + optional write applying new value. Per-attribute callbacks:
```c
struct attribute {
    const char *name;
    umode_t mode;
};

struct kobj_attribute {
    struct attribute attr;
    ssize_t (*show)(struct kobject *kobj, struct kobj_attribute *attr, char *buf);
    ssize_t (*store)(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count);
};
```

Plus per-class extensions (`struct device_attribute`, `struct driver_attribute`, `struct bus_attribute`, etc.) wrapping `kobj_attribute`.

### Symlinks for cross-references

sysfs uses symlinks extensively to express "is-a" + "owns" relationships:
- `/sys/class/net/eth0` → `/sys/devices/pci0000:00/0000:00:1f.6/net/eth0`
- `/sys/bus/pci/devices/0000:00:1f.6` → `/sys/devices/pci0000:00/0000:00:1f.6`
- `/sys/class/block/sda` → `/sys/devices/.../sda`

Identical symlink topology so existing tooling traverses correctly.

### `uevent` netlink

Per-kobj `uevent` file + per-system NETLINK_KOBJECT_UEVENT broadcast:
```
ACTION=add|remove|change|move|online|offline|bind|unbind
DEVPATH=/devices/...
SUBSYSTEM=...
[per-attr KEY=VALUE...]
```

Used by udev for plug-and-play device handling. Wire format byte-identical so udev rules work.

### kernfs vs sysfs

`kernfs` is the underlying VFS layer that sysfs built on top of (since v3.14). cgroup-v2 + configfs also use kernfs. sysfs adds:
- kobject-aware lookup
- per-attribute show/store dispatch
- uevent integration

`kernfs` provides:
- Tree structure (`kernfs_node`)
- Active-path locking (concurrent reader vs. mutator)
- Per-node attributes
- SUPER_BLOCK + INODE management

## Tier-3 docs governed by this Tier-2

(Phase C will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `fs/sysfs/sysfs-core.md` | sysfs core: dir/file/group/symlink/mount (`fs/sysfs/*.c`) |
| `fs/sysfs/kernfs.md` | kernfs underlying layer (`fs/kernfs/*`) |
| `fs/sysfs/kobject.md` | kobject framework (`lib/kobject.c`) |
| `fs/sysfs/kobject-uevent.md` | uevent infrastructure (`lib/kobject_uevent.c`) |

## Compatibility outline (top-level)

- REQ-O1: Top-level `/sys/*` subdirectories (block / bus / class / dev / devices / firmware / fs / hypervisor / kernel / module / power) registered identically.
- REQ-O2: `struct kobject` + `struct kobj_type` + `struct kset` layouts byte-identical first cache-line.
- REQ-O3: Attribute file convention: per-attribute show/store callbacks; default permissions per `mode`.
- REQ-O4: Per-bus / per-class / per-device extension types (`device_attribute` / `driver_attribute` / `bus_attribute`) layouts byte-identical.
- REQ-O5: Cross-reference symlinks for `class` / `bus` / `block` aliases byte-identical to upstream.
- REQ-O6: `uevent` netlink wire format byte-identical (NETLINK_KOBJECT_UEVENT).
- REQ-O7: kernfs tree-structure + active-path locking + super_block management identical.
- REQ-O8: Per-Tier-3 child documents define per-component requirements.
- REQ-O9: Hardening: row-1 features applied; default-on configurable per `00-security-principles.md`.

## Acceptance Criteria (top-level)

- [ ] AC-O1: `ls /sys/` shows expected top-level dirs. (covers REQ-O1)
- [ ] AC-O2: `pahole struct kobject` + `kobj_type` + `kset` byte-identical. (covers REQ-O2)
- [ ] AC-O3: Per-attribute test: synthetic kernel-module registers a kobj_attribute; `cat` returns value; `echo` updates. (covers REQ-O3)
- [ ] AC-O4: udev-equivalent test: `udevadm monitor` shows uevents on hotplug; format byte-identical. (covers REQ-O6)
- [ ] AC-O5: Symlink test: `/sys/class/net/eth0` resolves to actual device path under `/sys/devices/`. (covers REQ-O5)
- [ ] AC-O6: kernfs concurrency: register/unregister 1000 attribute files with concurrent readers; no use-after-free. (covers REQ-O7)
- [ ] AC-O7: Per-Tier-3 ACs cumulatively cover sysfs ABI. (meta-AC)
- [ ] AC-O8: Hardening section per Tier-3 child docs is non-empty. (covers REQ-O9)

## Architecture (top-level)

Each Tier-3 child doc declares its own Rust module organization. Shared abstractions:

- `kernel::lib::kobject::Kobject` — `struct kobject`
- `kernel::lib::kobject::KobjType` — `struct kobj_type` ktype
- `kernel::lib::kobject::Kset` — `struct kset` containers
- `kernel::lib::kobject::Uevent` — kobject_uevent emitter
- `kernel::fs::kernfs::Node` — `struct kernfs_node`
- `kernel::fs::kernfs::Tree` — kernfs tree mgmt
- `kernel::fs::kernfs::ActivePath` — active-path locking
- `kernel::fs::sysfs::SysFs` — top-level fs registration
- `kernel::fs::sysfs::Attribute` — attribute file dispatch
- `kernel::fs::sysfs::Group` — `struct attribute_group` (named groups of attributes)

## Verification (top-level)

### Layer 1: Kani SAFETY proofs

Per Tier-3.

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/fs/kernfs_active_path.tla` | `fs/sysfs/kernfs.md` (proves: kernfs active-path-locking semantics — concurrent reader vs. mutator coexistence; no use-after-free during file removal) |

### Layer 3: invariant harnesses

Per Tier-3.

### Layer 4: functional correctness (opt-in)

- **kobject refcount soundness theorem** via Verus — proves: per-kobject refcount transitions UNINIT → INITED → REFCOUNT_OK → REFCOUNT_DEAD; freed only after refcount=0.

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-kobject + per-kernfs_node + per-attribute refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-kobject + per-kernfs_node slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed kobject + kernfs_node cleared (carry name strings + per-attribute data) | § Default-on configurable off |
| **CONSTIFY** | per-ktype + per-attribute_group instances `static const` | § Mandatory |

### Row-1 features consumed by this Tier-2

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, CONSTIFY**: see above
- **USERCOPY**: per-attribute show/store uses bound-checked accessors
- **SIZE_OVERFLOW**: per-attribute name-length + per-tree-depth arithmetic uses checked operators
- **KERNEXEC**: per-attribute dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hooks: `security_inode_permission` / `security_file_permission` per-file gate.
- Default useful GR-RBAC policy: deny per-attribute writes outside gradm-marked roles for sensitive paths (`/sys/power/state`, `/sys/kernel/livepatch/*`, `/sys/kernel/debug/*`, `/sys/module/*/parameters/*`).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none at Tier-2 — defer to per-Tier-3)

## Out of Scope

- Per-Tier-3 child docs (Phase C continuation)
- Per-bus + per-class device-driver implementations (cross-ref individual driver Tier-3s when added)
- 32-bit-only paths
- Implementation code
