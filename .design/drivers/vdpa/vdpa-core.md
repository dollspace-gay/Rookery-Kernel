# Tier-3: drivers/vdpa/{vdpa,vdpa_user/*}.c — vDPA bus + VDUSE userland framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/vdpa/vdpa-core.md
upstream-paths:
  - drivers/vdpa/vdpa.c
  - drivers/vdpa/vdpa_user/vduse_dev.c
  - drivers/vdpa/vdpa_user/iova_domain.c
  - include/linux/vdpa.h
  - include/uapi/linux/vdpa.h
  - include/uapi/linux/vduse.h
-->

## Summary

vDPA (virtio Data Path Acceleration) — a kernel framework that exposes a virtio-compatible data plane backed by hardware (or by a userland process via VDUSE) but plugs into either a kernel virtio driver (via `virtio_vdpa`) or a userspace virtio consumer (via `vhost-vdpa`). The point is "virtio in the guest, anything underneath", with the kernel mediating descriptor-ring layout, feature negotiation, IOTLB programming, and PASID/group/address-space isolation.

This Tier-3 covers `drivers/vdpa/vdpa.c` (~1600 lines: vDPA bus, vdpa_mgmtdev registry, generic-netlink control plane VDPA_CMD_*) and `drivers/vdpa/vdpa_user/{vduse_dev.c,iova_domain.c}` (~3200 lines combined: VDUSE — a userland vDPA-device implementation framework that exposes `/dev/vduse/control` for create/destroy + `/dev/vduse/<name>` for the per-device control + message channel, plus a SW IOTLB / IOVA domain with bounce buffers for guest DMA).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vdpa_device` | per-vDPA device: config_ops, map_ops, ngroups, nas, virtio device_id | `drivers::vdpa::Device` |
| `struct vdpa_mgmt_dev` | per-management device: parent, supported classes, dev_add/del | `drivers::vdpa::MgmtDev` |
| `struct vdpa_config_ops` | per-device vtable: get_vq_num_max, get/set_status, set_vq_addr, set_features, get_config, set_map, dma_map/unmap | `drivers::vdpa::ConfigOps` |
| `vdpa_set_status(vdev, status)` | virtio device-status setter (under cf_lock write) | `Device::set_status` |
| `__vdpa_alloc_device(parent, config, map, ngroups, nas, size, name, use_va)` | alloc + IDA + bus binding | `Device::alloc` |
| `_vdpa_register_device(vdev, nvqs)` | publish to vdpa bus (must hold vdpa_dev_lock) | `Device::register` |
| genl ops (`VDPA_CMD_MGMTDEV_GET`, `_DEV_NEW`, `_DEV_DEL`, `_DEV_GET`, `_DEV_CONFIG_GET`, `_DEV_VSTATS_GET`, `_DEV_ATTR_SET`) | userland control plane via `nlmsg_unicast` | `Nl::cmd_*` |
| `vdpa_dev_config_fill(vdev, msg, ...)` | reply builder for DEV_CONFIG_GET | `Nl::config_fill` |
| `struct vduse_dev` / `struct vduse_virtqueue` | VDUSE per-device + per-vq state | `drivers::vduse::Dev` / `Vq` |
| `vduse_dev_msg_xfer(dev, msg, timeout)` | kernel → userspace message round-trip via /dev/vduse/<name> | `Dev::msg_xfer` |
| `vduse_dev_ioctl(filp, cmd, arg)` | per-device control: VDUSE_IOTLB_GET_FD, _VQ_GET_INFO, _VQ_SETUP_KICKFD, _DEV_GET/SET_CONFIG, _DEV_INJECT_CONFIG_IRQ | `Dev::ioctl` |
| `vduse_iotlb_link/unlink/io(...)`  | IOTLB programming + msync from userland | `Dev::iotlb_*` |
| `struct vduse_iova_domain` (`iova_domain.c`) | SW IOTLB + bounce-buffer pool over per-device IOVA range | `drivers::vduse::IovaDomain` |
| `vhost-vdpa` consumer | separate Tier-3 — vdpa_device bound to `/dev/vhost-vdpa-N` | cross-ref `drivers/vhost/vdpa.md` |
| `virtio_vdpa` consumer | kernel virtio driver bound to vDPA device | cross-ref `drivers/vdpa/virtio_vdpa.md` (future) |

## Compatibility contract

REQ-1: vDPA bus type registered with name `"vdpa"`; per-device match always returns 1 (driver_override required); per-device probe sets 64-bit DMA mask, validates `get_vq_num_max ≥ get_vq_num_min`, calls driver probe.

REQ-2: Generic-netlink family `"vdpa"` (resv_start_op = VDPA_CMD_DEV_VSTATS_GET+1); `_DEV_NEW` requires CAP_NET_ADMIN; `_MGMTDEV_GET`, `_DEV_GET`, `_DEV_CONFIG_GET`, `_DEV_VSTATS_GET` support dumpit walks.

REQ-3: Per-vDPA-device has `ngroups` virtqueue groups + `nas` address-spaces; per-group assigned to an AS at any moment; IOTLB programming is per-AS, isolating DMA across groups.

REQ-4: Per-device feature negotiation: `set_features(vdev, features)` rejects features not in `get_features(vdev)`; `features_valid` set true on first set; status DRIVER_OK gates virtqueue access.

REQ-5: Per-config-space access via `get_config(vdev, offset, buf, len)` / `set_config(...)`; `get_generation` provides config-update epoch (virtio 1.x semantic).

REQ-6: VDUSE control device `/dev/vduse/control` exposes ioctls: `VDUSE_GET_API_VERSION`, `_SET_API_VERSION`, `_CREATE_DEV`, `_DESTROY_DEV`. `CREATE_DEV` requires CAP_SYS_ADMIN AND module unloads gated by no-active-devices.

REQ-7: VDUSE per-device `/dev/vduse/<name>` exposes ioctls: `VDUSE_IOTLB_GET_FD`, `_DEV_GET/SET_CONFIG`, `_DEV_INJECT_CONFIG_IRQ`, `_VQ_GET_INFO`, `_VQ_SETUP_KICKFD`, `_VQ_INJECT_IRQ`, plus read/poll for kernel→userspace messages.

REQ-8: VDUSE per-device API version pinned per session: VDUSE_API_VERSION_NOT_ASKED (0) preserved for backward compat; new sessions migrated to current version on first VDUSE_SET_API_VERSION.

REQ-9: IOTLB per-AS programmed via VDUSE_IOTLB_GET_FD (returns an fd backed by `vhost_iotlb`); userspace mmaps + writes IOTLB entries; kernel validates each entry against AS bounds + permissions on use.

REQ-10: VDUSE bounce-buffer size per-device bounded between `VDUSE_MIN_BOUNCE_SIZE` (1MB) and `VDUSE_MAX_BOUNCE_SIZE` (1GB); IOVA range `VDUSE_IOVA_SIZE = VDUSE_MAX_BOUNCE_SIZE + 128MB`.

REQ-11: Group counts bounded: `VDUSE_DEV_MAX_GROUPS` = 0xffff, `VDUSE_DEV_MAX_AS` = 0xffff; per-device max `1 << MINORBITS` instances.

REQ-12: virtio device_id allowlist enforced — VDUSE refuses to create devices for IDs outside the supported set (VIRTIO_ID_NET, VIRTIO_ID_BLOCK initially; per-version-pin enumeration).

## Acceptance Criteria

- [ ] AC-1: `vdpa dev show` lists devices created via netlink; matches sysfs `/sys/bus/vdpa/devices/`.
- [ ] AC-2: vhost-vdpa attach to a vDPA device: qemu `-netdev type=vhost-vdpa,vhostdev=/dev/vhost-vdpa-0` boots guest with virtio-net.
- [ ] AC-3: virtio-vdpa attach: kernel virtio_net driver binds to vdpa0 → eth interface present.
- [ ] AC-4: VDUSE create-destroy cycle: `VDUSE_CREATE_DEV` then `VDUSE_DESTROY_DEV` leaves no leaked vdpa_devices.
- [ ] AC-5: VDUSE-backed virtio-blk reachable through vhost-vdpa from qemu guest with userland-disk-server backing.
- [ ] AC-6: IOTLB programming via VDUSE_IOTLB_GET_FD; guest DMA correctly routes through bounce buffers; reject IOVA outside declared AS range.
- [ ] AC-7: Feature negotiation: guest virtio driver acks only features the device offered; mismatched feature-set rejected at SET_FEATURES.
- [ ] AC-8: CAP_NET_ADMIN required for VDPA_CMD_DEV_NEW; CAP_SYS_ADMIN required for VDUSE_CREATE_DEV.

## Architecture

`Device` lives in `drivers::vdpa::Device`:

```
struct Device {
  dev: KernelDevice,
  index: u32,
  config: &'static dyn ConfigOps,
  map: Option<&'static dyn MapOps>,
  features_valid: AtomicBool,
  ngroups: u32,
  nas: u32,
  nvqs: u32,
  cf_lock: RwSemaphore,
  use_va: bool,
  refcount: Refcount,
}

struct MgmtDev {
  list: ListHead,
  parent: Arc<KernelDevice>,
  classes: Vec<&'static dyn MgmtDevClass>,
  dev_add: fn(...),
  dev_del: fn(...),
  supported_features: u64,
}
```

vDPA bus init `Subsystem::init`:
1. `bus_register(&vdpa_bus)`.
2. `genl_register_family(&vdpa_nl_family)`.
3. Module per backend (mlx5_vdpa, ifcvf, vdpa_sim, vduse) registers a `vdpa_mgmt_dev` via `vdpa_mgmtdev_register`.

Device create via netlink `Nl::cmd_dev_add`:
1. Parse `VDPA_ATTR_MGMTDEV_BUS_NAME` + `_DEV_NAME`.
2. CAP_NET_ADMIN gate.
3. `vdpa_mgmtdev_get_from_attr` — locate the management device.
4. `mgmt->dev_add(mgmt, name, attrs)` → backend allocs + registers `vdpa_device`.
5. Bus probe path runs (`vdpa_dev_probe`); driver_override picks consumer (vhost-vdpa / virtio-vdpa).

VDUSE-backed device flow:
1. Userspace opens `/dev/vduse/control`, sets API version, calls `VDUSE_CREATE_DEV` → kernel allocates `vduse_dev`, registers misc cdev `/dev/vduse/<name>`.
2. Userspace opens `/dev/vduse/<name>`, calls `VDUSE_IOTLB_GET_FD` to bind an IOTLB fd per-AS, calls `VDUSE_VQ_SETUP_KICKFD` per-vq.
3. Userspace then asks `vdpa dev add mgmtdev vduse name <name>` via netlink → `vduse_mgmtdev_dev_add` allocates the matching `vdpa_device`, wiring its `config_ops` so reads/writes round-trip through the user device via msg channel.
4. Kernel virtio / vhost-vdpa consumer binds and starts driving the vDPA device; every config/status/feature touch translates into a message on `recv_list` consumed by userland read().
5. Guest DMA: device emits IOVA; `vduse_iova_domain` translates via IOTLB; if mapping missing or IOMMU-translated to bounce, copy through bounce-buffer pool.

IOVA domain `vduse_iova_domain` (iova_domain.c):
- Per-domain bitmap of bounce-buffer pages + LRU/free-list.
- Per-IOVA lookup uses `vhost_iotlb` interval tree.
- `vduse_domain_map_page` / `_unmap_page` provide dma_map_ops equivalents for userland-DMA.

cf_lock semantics: `vdpa_set_status` and `vdpa_set_features` take cf_lock for-write; `get_config` / `get_status` take it for-read; defends against concurrent consumer race.

## Hardening

(Inherits from `drivers/vdpa/00-overview.md` § Hardening.)

vdpa-core specific reinforcement:

- **Per-device IDA bounded** — `vdpa_index_ida` allocation prevents index reuse during outstanding refcount.
- **MgmtDev list rwlock-protected** — `vdpa_dev_lock` rwsem; reads during dump, writes during dev_add/del.
- **VDUSE bounce pool size validated** — per-device bounce size in `[VDUSE_MIN_BOUNCE_SIZE, VDUSE_MAX_BOUNCE_SIZE]`.
- **VDUSE msg send-list bounded** — per-device `send_list` length capped to prevent userland-stall DoS.
- **VDUSE `broken` flag latched on protocol violation** — once tripped, all subsequent ops return -EIO.
- **vhost_iotlb entries validated against AS bounds** — IOVA + size ≤ declared AS limit; reject otherwise.
- **API-version pin per session** — refuse downgrade; `VDUSE_API_VERSION_NOT_ASKED` migration is one-way.
- **`features_valid` latch** — once set true, future `set_features` calls rejected unless device reset to status=0.
- **`use_va` constrained** — requires `dma_map || set_map`; otherwise IOTLB cannot be programmed safely.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `vdpa_device`, `vdpa_mgmt_dev`, `vduse_dev`, `vduse_virtqueue` slabs; bounce-buffer pool whitelisted; reject userland copy that crosses IOTLB-bound boundaries.
- **PAX_KERNEXEC** — vDPA bus ops (`probe`, `remove`, `match`), `vdpa_config_ops` vtables, and netlink doit/dumpit handlers live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `vdpa_nl_cmd_dev_*`, `vduse_dev_ioctl`, IOTLB programming, and msg-xfer entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `vdpa_device`, `vduse_dev` (`vduse_dev_get/_put`), `vduse_iova_domain`, and per-vq state; overflow trap defeats VDUSE-create/destroy + consumer-attach races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for bounce-buffer pages, IOTLB entries, vq descriptor caches, config-space backing memory; defense against config-leak across device-id reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every ioctl/netlink/read/write/mmap entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `vdpa_config_ops` (`set_vq_addr`, `set_status`, `set_features`, `get_config`, `set_map`, `dma_map`, `dma_unmap`), `vdpa_map_ops`, and genl_ops `.doit`/`.dumpit` marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — suppress `%p` of `vdpa_device`, `vduse_dev`, vq pointers, IOTLB entries in tracepoints; gate kallsyms behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict VDUSE protocol-violation, IOTLB-rejection, feature-mismatch banners to CAP_SYSLOG; suppress device-id + addr disclosure.
- **CAP_SYS_ADMIN strict** — `/dev/vhost-vdpa-*` open and VDUSE_CREATE_DEV / _DESTROY_DEV require CAP_SYS_ADMIN in the owning user namespace.
- **CAP_NET_ADMIN strict** — `VDPA_CMD_DEV_NEW`, `_DEV_DEL`, `_DEV_ATTR_SET` netlink ops require CAP_NET_ADMIN.
- **CAP_SYS_RAWIO** — `/dev/vhost-vdpa-*` open additionally gated by CAP_SYS_RAWIO so DMA-programming surfaces are not delegated to non-RAWIO callers.
- **IOTLB strict mode** — refuse identity mapping; every IOTLB entry validated against declared AS bounds + permissions; entries persist only while consumer holds matching reference.
- **virtio-feature allowlist** — per-mgmt-device declares supported features; `set_features` refuses bits outside that allowlist (including legacy/experimental bits).
- **Device-id PAX_REFCOUNT** — virtio device_id and PASID refcount overflow-protected; refuse re-bind while in-flight DMA.
- **Name allowlist** — VDUSE device names validated to printable-ASCII, length-bounded, refusing path separators and control chars; defense against udev-rule injection.

Rationale: vDPA is virtio-from-anywhere — VDUSE turns a userland process into a DMA-capable virtio device. A loose IOTLB policy, a permissive feature negotiation, an unbounded msg queue, or a refcount underflow on `vduse_dev` lets userland (or a malicious mgmt-device) inject arbitrary DMA into guest memory, race consumer attach, or pivot from net-admin to host kernel memory. RAP/kCFI on `vdpa_config_ops`, CAP_NET_ADMIN + CAP_SYS_ADMIN gating, IOTLB strict bounds, feature/device-id allowlists, and refcount-overflow trap turn vDPA into a structural isolation boundary rather than a courtesy API.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- vhost-vdpa consumer (covered in `drivers/vhost/vdpa.md`)
- virtio-vdpa kernel consumer (future Tier-3)
- mlx5_vdpa / ifcvf / pds_vdpa hardware backends (separate Tier-3s)
- vDPA simulator (vdpa_sim) (future Tier-3)
- IOMMUFD integration for vDPA (future)
- Implementation code
