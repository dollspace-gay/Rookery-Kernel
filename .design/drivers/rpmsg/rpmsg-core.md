# Tier-3: drivers/rpmsg/{rpmsg_core,rpmsg_char,virtio_rpmsg_bus}.c — rpmsg bus, endpoints, char-dev + virtio backend

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/rpmsg/rpmsg-core.md
upstream-paths:
  - drivers/rpmsg/rpmsg_core.c
  - drivers/rpmsg/rpmsg_char.c
  - drivers/rpmsg/rpmsg_ctrl.c
  - drivers/rpmsg/rpmsg_ns.c
  - drivers/rpmsg/rpmsg_internal.h
  - drivers/rpmsg/virtio_rpmsg_bus.c
  - include/linux/rpmsg.h
  - include/uapi/linux/rpmsg.h
-->

## Summary

rpmsg — the kernel's "remote-processor messaging" bus, sitting above remoteproc's `vdev` resource entries. It provides bidirectional channels (`rpmsg_device`) carrying named services between the host CPU and an attached co-processor (DSP, Cortex-M, modem, etc.), implemented over a shared-memory virtio-ring transport (`virtio_rpmsg_bus.c`) or an SoC-specific transport (Qualcomm SMD / GLINK, MediaTek mtk_rpmsg). Per-rpmsg channel is a `<name, src_addr, dst_addr>` tuple; per-endpoint binds an RX callback to an address; the bus matches `rpmsg_driver` against channel names + ids.

This Tier-3 covers `rpmsg_core.c` (~670 lines: bus registration, ept create/destroy, send/sendto/trysend, name-service helpers) + `rpmsg_char.c` (~570 lines: `/dev/rpmsg<N>.<chname>.<rsrc>.<rdst>` per-channel char device for userspace rpmsg) + transport summary from `virtio_rpmsg_bus.c` (~1000 lines: virtio-ring backend, default-channel announce, NS service). `rpmsg_ctrl.c` provides the `/dev/rpmsg_ctrl<N>` ept creation cdev.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rpmsg_device` | per-channel: name, src/dst addr, ept, ops, announce flag | `drivers::rpmsg::Device` |
| `struct rpmsg_endpoint` | per-endpoint: cb, priv, src addr, parent rpdev, ops | `drivers::rpmsg::Endpoint` |
| `struct rpmsg_channel_info` | name (RPMSG_NAME_SIZE=32) + src/dst (`u32`) | `drivers::rpmsg::ChannelInfo` |
| `struct rpmsg_device_ops` | per-transport vtable: create_channel, release_channel, create_ept, announce_create/destroy | `drivers::rpmsg::DeviceOps` |
| `struct rpmsg_endpoint_ops` | per-ept vtable: destroy_ept, send, sendto, send_offchannel, trysend(to), get_mtu | `drivers::rpmsg::EndpointOps` |
| `rpmsg_register_device(rpdev)` / `_unregister_device(...)` | bus-level publish/withdraw | `Device::register` / `unregister` |
| `rpmsg_create_channel(rpdev, chinfo)` / `_release_channel(...)` | per-transport channel mgmt | `Device::create_channel` / `release_channel` |
| `rpmsg_create_ept(rpdev, cb, priv, chinfo)` / `_destroy_ept(ept)` | per-endpoint life | `Endpoint::create` / `destroy` |
| `rpmsg_send(ept, data, len)` / `_sendto(..., dst)` / `_trysend(...)` / `_send_offchannel(...)` | send variants | `Endpoint::send_*` |
| `rpmsg_get_mtu(ept)` | per-endpoint MTU | `Endpoint::mtu` |
| `rpmsg_class` (`/sys/class/rpmsg`) | sysfs class for channel discovery | `Subsystem::class` |
| `rpmsg_chrdev_eptdev_destroy(dev, data)` | char-dev teardown when underlying rpdev dies | `Char::eptdev_destroy` |
| `struct rpmsg_eptdev` | per-`/dev/rpmsg<N>` cdev: rpdev, chinfo, ept, queue, readq | `drivers::rpmsg::Eptdev` |
| ioctls `RPMSG_DESTROY_EPT_IOCTL`, `RPMSG_GET_OUTGOING_FLOWCONTROL`, `RPMSG_SET_OUTGOING_FLOWCONTROL` | per-cdev control | `Char::ioctl` |
| `/dev/rpmsg_ctrl<N>` (`rpmsg_ctrl.c`) | userland-side ept creation: `RPMSG_CREATE_EPT_IOCTL` | `drivers::rpmsg::Ctrl` |
| `virtio_rpmsg_bus.c` | virtio backend: per-vdev pair of vqs (RX/TX) + default-channel announce + ns service | `drivers::rpmsg::VirtioBackend` |
| `rpmsg_ns.c` | name-service announce/withdraw protocol | `drivers::rpmsg::NameService` |

## Compatibility contract

REQ-1: rpmsg bus registered with name `"rpmsg"`; bus match either by name (`rpmsg_match_device`) or by `rpmsg_device_id` table; uevent emits `MODALIAS=rpmsg:<name>`.

REQ-2: Per-channel `(name, src, dst)` tuple is unique within a transport; channel names ≤ RPMSG_NAME_SIZE (32 bytes, NUL-terminated).

REQ-3: Address space: 32-bit `src`/`dst` addresses; `RPMSG_ADDR_ANY` (0xFFFFFFFF) requests dynamic assignment; reserved low addresses (RPMSG_NS_ADDR=53) for name-service.

REQ-4: Per-endpoint RX callback signature `int (*rpmsg_rx_cb_t)(struct rpmsg_device *, void *buf, int len, void *priv, u32 src)`; return code negative aborts buffer release.

REQ-5: Per-channel default endpoint created automatically when `rpmsg_register_device` is called; destroyed when device removed; additional endpoints created on demand via `rpmsg_create_ept`.

REQ-6: virtio backend: per-vdev has two virtqueues — RX (kernel-consumed) + TX (kernel-produced); default buffer size 512 bytes; per-buffer headers `struct rpmsg_hdr` (src + dst + reserved + len + flags + payload).

REQ-7: Name-service protocol (`rpmsg_ns.c`): RPMSG_NS_CREATE / RPMSG_NS_DESTROY messages on reserved address 53 announce channels created by the remote; host instantiates matching `rpmsg_device`.

REQ-8: `/dev/rpmsg_ctrl<N>` ioctl `RPMSG_CREATE_EPT_IOCTL` allows userspace to create endpoint bound to arbitrary `(name, src, dst)`; resulting `/dev/rpmsg<N>.<name>.<src>.<dst>` cdev presents read/write/poll.

REQ-9: `/dev/rpmsg<N>.<...>` per-cdev: blocking read consumes one message; write sends to the channel default dst (or via flow-control); poll signals `POLLIN | POLLPRI` on flow-update.

REQ-10: Per-endpoint MTU exposed via `rpmsg_get_mtu(ept)`; userland sees `RPMSG_GET_OUTGOING_FLOWCONTROL` for flow status.

REQ-11: Per-cdev sk_buff queue bounded; messages received under spinlock and woken via `readq` wait-queue.

REQ-12: Bus lifetime tied to underlying transport: when rproc shuts down, all rpdev are unregistered; userland reads return `-ENETRESET` until reconnect.

## Acceptance Criteria

- [ ] AC-1: After remoteproc boot of an rpmsg-aware firmware, `ls /sys/class/rpmsg/` shows at least one `rpmsg<N>` device per advertised channel.
- [ ] AC-2: `/dev/rpmsg_ctrl<N>` open + `RPMSG_CREATE_EPT_IOCTL` produces `/dev/rpmsg<N>.<name>.<src>.<dst>`.
- [ ] AC-3: Userland write + remote echo back is observable via blocking read on the cdev within bounded latency.
- [ ] AC-4: `rpmsg_send` MTU enforced — over-MTU payload returns `-EMSGSIZE`.
- [ ] AC-5: Channel withdraw: remote sends RPMSG_NS_DESTROY → `rpmsg_eptdev` reports `-ENETRESET` on read; cdev removed on close.
- [ ] AC-6: Flow control: remote sets restrict → cdev poll wakes; `RPMSG_GET_OUTGOING_FLOWCONTROL` reports current state.
- [ ] AC-7: Per-endpoint destroy on cdev close: subsequent NS_CREATE for the same name re-instantiates cleanly with no leaked addresses.
- [ ] AC-8: virtio-rpmsg buffer reuse correct across many send/recv cycles; no missed buffer-return after `rpmsg_destroy_ept`.

## Architecture

`Device` lives in `drivers::rpmsg::Device`:

```
struct Device {
  dev: KernelDevice,
  id: RpmsgDeviceId,         // {name[32], driver_data}
  desc: KString,
  src: u32,
  dst: u32,
  ept: Option<Arc<Endpoint>>,   // default endpoint
  announce: bool,                // emit NS create/destroy on bus add/remove
  little_endian: bool,
  ops: &'static dyn DeviceOps,
}

struct Endpoint {
  refcount: Refcount,
  rpdev: Weak<Device>,
  cb: RxCb,
  flow_cb: Option<FlowCb>,
  cb_lock: Mutex<()>,
  addr: u32,
  priv: KBox<dyn Any>,
  ops: &'static dyn EndpointOps,
}

struct Eptdev {
  dev: KernelDevice,
  cdev: Cdev,
  rpdev: Weak<Device>,
  chinfo: ChannelInfo,
  ept_lock: Mutex<()>,
  ept: Option<Arc<Endpoint>>,
  default_ept: Option<Arc<Endpoint>>,
  queue_lock: SpinLock<()>,
  queue: SkbQueue,
  readq: WaitQueueHead,
  remote_flow_restricted: bool,
  remote_flow_updated: bool,
}
```

Bus init `Subsystem::init`:
1. `class_register(&rpmsg_class)` — `/sys/class/rpmsg`.
2. `bus_register(&rpmsg_bus)` — bus type `"rpmsg"`.
3. `alloc_chrdev_region(&rpmsg_major, 0, RPMSG_DEV_MAX, "rpmsg")`.
4. Backend modules (`virtio_rpmsg_bus`, `qcom_smd`, `qcom_glink_*`, `mtk_rpmsg`) register their per-transport probe paths.

Per-channel registration `Device::register`:
1. `dev->bus = &rpmsg_bus`; `dev->release = rpmsg_release_device` (or override).
2. `device_add(&dev)` triggers `rpmsg_dev_probe`:
   a. Resolve driver via `rpmsg_dev_match` (id table or name).
   b. Allocate default endpoint via `rpdev->ops->create_ept(rpdev, drv->callback, NULL, chinfo)`.
   c. Call `drv->probe(rpdev)`.
   d. If `announce`: `rpdev->ops->announce_create(rpdev)` sends NS_CREATE.

Per-endpoint life `Endpoint::create`:
1. `rpdev->ops->create_ept(rpdev, cb, priv, chinfo)` — backend allocates ept ID via per-transport `idr`/`ida`.
2. For dynamic addresses (`RPMSG_ADDR_ANY`), backend allocates from per-transport pool above reserved range.
3. Endpoint registered in `rpdev`'s ept table so inbound messages are routed to the right `cb`.

Send `Endpoint::send`:
1. `ept->ops->send(ept, data, len)`.
2. virtio backend path: locks TX virtqueue, picks free TX buffer, fills `rpmsg_hdr` + payload, `virtqueue_kick(tx_vq)`.
3. Returns 0 or `-EMSGSIZE` (over-MTU), `-ENXIO` (no transport), `-ERESTARTSYS` (15s block timeout).

Receive (virtio backend): RX virtqueue interrupt → `virtio_rpmsg_recv_done` → per-buffer parse → look up dst address in ept idr → call `ept->cb(rpdev, buf, len, priv, src)`; buffer returned to vq after cb.

Char-dev surface `Eptdev`:
- `open`: alloc/bind ept via `rpmsg_create_ept(rpdev, rpmsg_ept_cb, eptdev, chinfo)` (or default-ept reuse if `default_ept` set); install `flow_cb = rpmsg_ept_flow_cb`.
- `read`: dequeue skb from `eptdev->queue`, copy to user.
- `write`: `rpmsg_send` (or `rpmsg_sendto` if alt dst supplied via ioctl).
- `ioctl`: `RPMSG_DESTROY_EPT_IOCTL` (per-cdev ept destroy), `RPMSG_GET_OUTGOING_FLOWCONTROL`, `RPMSG_SET_OUTGOING_FLOWCONTROL`.
- `poll`: `POLLIN` when queue non-empty, `POLLPRI` when `remote_flow_updated`.
- `release`: destroy ept (unless `default_ept`), purge queue, drop ref.

`/dev/rpmsg_ctrl<N>` (rpmsg_ctrl.c):
- `RPMSG_CREATE_EPT_IOCTL` (RPMSG_CREATE_EPT_IOCTL): takes `struct rpmsg_endpoint_info { name[32], src, dst }`; calls `rpmsg_chrdev_eptdev_create` to spawn `/dev/rpmsg<N>.<name>.<src>.<dst>`.

## Hardening

(Inherits from `drivers/rpmsg/00-overview.md` § Hardening.)

rpmsg-core specific reinforcement:

- **Per-transport address allocator bitmap-bounded** — RPMSG_ADDR_ANY allocations from `idr` are bounded to declared transport range.
- **Per-channel `RPMSG_NAME_SIZE` truncation enforced** — refuse channel names with embedded NUL or length > 31.
- **MTU enforcement on send** — `len > rpmsg_get_mtu(ept)` returns `-EMSGSIZE`.
- **Per-cdev skb queue length bounded** — defense against remote-side flooding eating host kernel memory.
- **NS service address (53) reserved** — refuse user-cdev creation of an endpoint on the NS address.
- **`announce_create`/`destroy` are atomic vs cdev open** — per-rpdev ept_lock prevents read/write on a destroyed channel.
- **Backend ept-table walk under per-rpdev lock** — concurrent destroy + RX race eliminated.
- **15s send block timeout** — `rpmsg_send` cannot indefinitely stall on TX-buffer-exhaustion.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `rpmsg_device`, `rpmsg_endpoint`, `rpmsg_eptdev`, virtio RX/TX buffer pools; endpoint addr fields validated before copy_to_user in ioctls.
- **PAX_KERNEXEC** — `rpmsg_device_ops`, `rpmsg_endpoint_ops`, bus `match`/`probe`/`remove`, and cdev `fops` live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `rpmsg_dev_probe`, send/recv, eptdev open/read/write/ioctl, NS callback entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `rpmsg_endpoint`, `rpmsg_eptdev` device refs, NS-service refs; overflow trap defeats channel-announce + cdev-close UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for RX/TX buffer pools, eptdev sk_buff caches, name-service message scratch; defense against payload-leak across endpoint reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every cdev ioctl/read/write entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `rpmsg_device_ops` (`create_channel`, `release_channel`, `create_ept`, `announce_create`, `announce_destroy`), `rpmsg_endpoint_ops` (`destroy_ept`, `send`, `sendto`, `send_offchannel`, `trysend*`, `get_mtu`), and cdev `file_operations` marked `__ro_after_init` with kCFI typed dispatch.
- **GRKERNSEC_HIDESYM** — suppress `%p` of `rpmsg_device`, endpoint, virtio vq pointers in tracepoints; gate kallsyms behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict NS-protocol-violation, MTU-reject, address-exhaustion banners to CAP_SYSLOG.
- **CAP_SYS_ADMIN on `/dev/rpmsg_ctrl*`** — `RPMSG_CREATE_EPT_IOCTL` (which can target arbitrary `(name, src, dst)`) gated by CAP_SYS_ADMIN in the owning user namespace.
- **Endpoint address PAX_USERCOPY whitelist** — `rpmsg_endpoint_info` userland struct validated for ASCII name + reserved-address rejection before processing.
- **Virtio-backend ID gate** — `virtio_rpmsg_bus` probe requires `VIRTIO_ID_RPMSG`-matching device-id; refuse other virtio IDs from claiming the rpmsg transport.
- **Channel name allowlist** — channel names validated against printable-ASCII charset, length-bounded ≤ RPMSG_NAME_SIZE-1, refusing path separators / NUL embeds.
- **Per-cdev udev rule alignment** — `/dev/rpmsg<N>.<name>.<src>.<dst>` nodes created mode 0660 with `rpmsg` group; refuse world-rw default permissions.

Rationale: rpmsg is a foreign-CPU/peripheral-CPU IPC channel — the remote side is firmware that may have been independently compromised (cf. baseband / DSP exploits). A loose endpoint allocator, missing CAP gate on `/dev/rpmsg_ctrl*`, unbounded skb queue, or refcount underflow on an `rpmsg_endpoint` becomes a pivot for the remote into host kernel memory or for unprivileged userland to talk to security-critical co-processors (secure-element, TEE, modem auth state). RAP/kCFI on the rpmsg-ops vtables, CAP_SYS_ADMIN on the ctrl device, virtio-id gating, channel-name allowlists, and refcount-overflow trapping turn rpmsg from a casual IPC into a structural boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Qualcomm SMD / GLINK backends (`qcom_smd.c`, `qcom_glink_*.c` — separate Tier-3s)
- MediaTek mtk_rpmsg backend (future Tier-3)
- remoteproc lifecycle (covered in `drivers/remoteproc/remoteproc-core.md`)
- rpmsg-tty userland framing (future Tier-3)
- Implementation code
