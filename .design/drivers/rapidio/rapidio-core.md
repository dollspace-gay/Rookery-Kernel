# Tier-3: drivers/rapidio/{rio,rio-driver,rio-sysfs,rio-scan,devices/rio_mport_cdev}.c — RapidIO subsystem core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/rapidio/rio.c
  - drivers/rapidio/rio-driver.c
  - drivers/rapidio/rio-sysfs.c
  - drivers/rapidio/rio-scan.c
  - drivers/rapidio/rio-access.c
  - drivers/rapidio/rio_cm.c
  - drivers/rapidio/devices/rio_mport_cdev.c
  - drivers/rapidio/devices/tsi721.c
  - drivers/rapidio/switches/idt_gen2.c
  - include/linux/rio.h
  - include/linux/rio_drv.h
  - include/linux/rio_regs.h
  - include/uapi/linux/rio_mport_cdev.h
  - include/uapi/linux/rio_cm_cdev.h
-->

## Summary

RapidIO is a packet-switched embedded interconnect (RapidIO Trade Association) used in radar, telecom backplane, and DSP-farm systems — Freescale QorIQ + IDT Tsi721 PCIe-to-SRIO bridges are the canonical platforms. The Linux RapidIO subsystem provides a `rio_bus_type` device model plus a per-master-port (mport) device class, scan helpers that enumerate the SRIO fabric (endpoint + switch discovery via maintenance read/write of the SRIO Common Transport / Configuration registers), the four traditional SRIO messaging primitives (Doorbells / Mailbox-messages / RapidIO-DMA / Port-write events), and the `/dev/rio_mport_cdevN` character interface for userspace clients (the "rio_mport_cdev" cdev). A companion `/dev/rio_cm` cdev (`rio_cm.c`) provides a connection-oriented (CM-style) socket abstraction on top.

The subsystem is layered: per-mport host (`tsi721.c`, MPC85xx fsl_rio) drives the hardware bridge and registers a `struct rio_mport` with `rio_register_mport`. Per-mport scan discovers `rio_dev` endpoints + `rio_switch` switches by walking maintenance config space (`rio-scan.c`) and installing routing tables on switches. Per-endpoint driver matches via `rio_driver.id_table` on `vid/did/asm_vid/asm_did`. Userland gets `/dev/rio_mport_cdevN` for maintenance R/W, doorbells, outbound DMA, inbound-DMA buffer registration, and CM connection mgmt.

This Tier-3 covers `rio.c` (~2150 lines: subsystem core, message/doorbell/pwrite resource allocators, inb/outb mapping), `rio-driver.c` (~270 lines: `rio_bus_type` + driver register/match), `rio-sysfs.c` (~370 lines: per-rdev sysfs + maintenance-space `config` binfile), `rio-scan.c` (~1160 lines: fabric enumeration), and `devices/rio_mport_cdev.c` (the `/dev/rio_mport_cdev*` UAPI).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `rio_bus_type` | device-model bus | `drivers::rapidio::Bus` |
| `rio_mport_class` | per-mport class (`/sys/class/rio_mport/`) | `drivers::rapidio::MportClass` |
| `struct rio_mport` | per-master-port control block | `drivers::rapidio::Mport` |
| `struct rio_dev` | per-endpoint device record | `drivers::rapidio::RioDev` |
| `struct rio_net` | per-network (one per-mport, switches+endpoints) | `drivers::rapidio::Net` |
| `struct rio_switch` | per-switch sub-record (routing table) | `drivers::rapidio::Switch` |
| `struct rio_ops` | per-mport hardware op-vector (maint R/W, dbell send, mbox send/recv, dma, pwrite) | `drivers::rapidio::Ops` |
| `rio_register_mport(port)` / `_unregister_mport(port)` | mport register/unregister | `Mport::register` / `_unregister` |
| `rio_add_device(rdev)` / `rio_del_device(rdev, state)` | endpoint add/remove | `RioDev::add` / `_del` |
| `rio_alloc_net(mport)` / `rio_add_net(net)` / `rio_free_net(net)` | per-mport network mgmt | `Net::alloc` / `_add` / `_free` |
| `rio_local_get_device_id(port)` / `rio_local_set_device_id(port, did)` | mport's own destination id | `Mport::local_did` |
| `rio_request_inb_mbox(mport, dev_id, mbox, entries, cb)` / `rio_release_inb_mbox(...)` | inbound mailbox alloc | `Mport::request_inb_mbox` / `_release_inb_mbox` |
| `rio_request_outb_mbox(...)` / `rio_release_outb_mbox(...)` | outbound mailbox alloc | `Mport::request_outb_mbox` / `_release_outb_mbox` |
| `rio_request_inb_dbell(mport, dev_id, start, end, cb)` / `_release_inb_dbell(...)` | inbound doorbell range alloc | `Mport::request_inb_dbell` / `_release_inb_dbell` |
| `rio_request_outb_dbell(rdev, start, end)` / `_release_outb_dbell(rdev, res)` | outbound doorbell range alloc | `RioDev::request_outb_dbell` / `_release_outb_dbell` |
| `rio_add_mport_pw_handler(mport, ctx, cb)` / `_del_mport_pw_handler(...)` | port-write event handler | `Mport::add_pw_handler` |
| `rio_request_inb_pwrite(rdev, cb)` / `_release_inb_pwrite(rdev)` | per-endpoint port-write delivery | `RioDev::request_inb_pwrite` |
| `rio_pw_enable(mport, enable)` | enable/disable port-write reception | `Mport::pw_enable` |
| `rio_map_inb_region(mport, local, rbase, sz, flags)` / `_unmap_inb_region(...)` | inbound DMA window | `Mport::map_inb_region` |
| `rio_map_outb_region(mport, destid, rbase, sz, rflags, &laddr)` / `_unmap_outb_region(...)` | outbound DMA window | `Mport::map_outb_region` |
| `rio_mport_get_physefb(...)` / `rio_set_port_lockout(...)` / `rio_get_comptag(...)` | fabric helpers | `Mport::*` |
| `rio_scan_alloc_net(mport, do_enum, name)` (in `rio-scan.c`) | enumerate fabric | `Mport::scan` |
| `rio_register_driver(rdrv)` / `_unregister_driver(rdrv)` | endpoint driver register | `Driver::register` / `_unregister` |
| `rio_match_bus(dev, drv)` | bus match by `rio_device_id` | `Bus::match` |
| `rio_read_config(file, kobj, attr, buf, off, count)` / `_write_config(...)` (in `rio-sysfs.c`) | per-rdev maint-space binfile | `RioDev::sysfs_config` |
| `mport_cdev_ioctl` (in `rio_mport_cdev.c`) | UAPI dispatch | `MportCdev::ioctl` |

## Compatibility contract

REQ-1: ABI `include/linux/rio.h` for in-kernel + `include/uapi/linux/rio_mport_cdev.h` + `include/uapi/linux/rio_cm_cdev.h` UAPI must remain stable: `RIO_MPORT_*` ioctls (`MAINT_HDID_SET`, `MAINT_COMPTAG_SET`, `MAINT_READ`, `MAINT_WRITE`, `DEV_ADD`, `DEV_DEL`, `OBW_ENABLE`, `OBW_DISABLE`, `OBW_FREE_MAP`, `OBW_MAP_REQUEST`, `IBW_ENABLE`, `IBW_DISABLE`, `IBW_MAP_REQUEST`, `IBW_FREE_MAP`, `EVENT_RD`, `EVENT_WR`, `XFER`, `DMA_FREE_HANDLE`, `WAIT_FOR_ASYNC`).

REQ-2: SRIO Common Transport — 16-bit destination ids (`u16 destid`), 8-bit hop count for maintenance access, 5-bit transport-type, 32-bit component tag; per-spec packet types Type 5 (NWRITE), Type 6 (SWRITE), Type 8 (MAINT R/W), Type 9 (data streaming), Type 10 (doorbell), Type 11 (message), Type 13 (response).

REQ-3: Per-mport scan: depth-first enumeration of switches + endpoints via Type 8 maintenance read of CAR/CSR registers; routing-table programming on each switch hop; mport-specific `rio_ops.lcread` / `_lcwrite` (local config) and `_cread` / `_cwrite` (network maint).

REQ-4: Per-mport messaging engine — each `rio_mport` exposes up to `RIO_MAX_MBOX = 4` inbound mailboxes + 4 outbound mailboxes; per-mbox callback `inb_msg_cb` / `outb_msg_cb` registered by an endpoint driver.

REQ-5: Per-mport doorbell ranges — each mport supports up to `RIO_MAX_INB_DOORBELLS` inbound dbell entries (range `[start, end)`); per-range callback delivers on arrival.

REQ-6: Per-endpoint port-write — an SRIO endpoint can asynchronously deliver a 64-byte port-write packet; the subsystem fans these out to per-endpoint handlers registered via `rio_request_inb_pwrite` and per-mport global handlers via `rio_add_mport_pw_handler`.

REQ-7: Per-mport DMA — inbound/outbound regions mapped via `rio_map_inb_region` / `rio_map_outb_region` with `RIO_MAP_FLAG_RFLOW` etc.; per-mport `rio_ops` supplies the hardware mapper.

REQ-8: `/dev/rio_mport_cdevN` per-mport char device (UAPI in `rio_mport_cdev.h`): opens require CAP_SYS_RAWIO; maintenance R/W ioctls validate `RIO_MAINT_SPACE_SZ = 16 MiB` bound on offset+size.

REQ-9: `/dev/rio_cm` (UAPI in `rio_cm_cdev.h`): connection-oriented socket abstraction (`RIO_CM_EP_CREATE`, `_LISTEN`, `_ACCEPT`, `_CONNECT`, `_SEND`, `_RECEIVE`, `_CLOSE`); built on top of the messaging engine.

REQ-10: `rio_register_driver` matches `rio_device_id { vid, did, asm_vid, asm_did }`; `RIO_ANY_ID = 0xffff` wildcards.

REQ-11: Per-rdev sysfs binfile `/sys/bus/rio/devices/.../config` (and `/sys/class/rio_mport/rio_mportN/maint`): config-space window into the device's maintenance space; R/W requires CAP_SYS_RAWIO.

REQ-12: `rio_mport_cdev` DMA-transfer ioctls validate per-XFER request struct + scatter-gather list size + per-mport DMA-channel capability; CAP_SYS_RAWIO required.

## Acceptance Criteria

- [ ] AC-1: Boot with Tsi721 PCIe-to-SRIO bridge attached; `/sys/class/rio_mport/rio_mport0/` populated; `/dev/rio_mport_cdev0` and `/dev/rio_cm` present.
- [ ] AC-2: Fabric scan enumerates a known-topology (1 mport + 1 IDT-CPS switch + 2 endpoints); `/sys/bus/rio/devices/` contains all four.
- [ ] AC-3: rio_mport_cdev maintenance read of endpoint's component-tag returns expected value.
- [ ] AC-4: Outbound doorbell range allocated, sent to peer, received via inbound dbell callback.
- [ ] AC-5: Outbound message via `rio_add_outb_message` round-trips through Tsi721 message engine.
- [ ] AC-6: rio_cm: server `LISTEN`+`ACCEPT` against client `CONNECT`; `SEND`/`RECEIVE` 1 MiB stream; no leak.
- [ ] AC-7: Hot-remove of mport (PCIe surprise removal of Tsi721): per-net rdevs torn down, per-mport pwrite disabled, no UAF.
- [ ] AC-8: rio_mport_cdev open from non-CAP_SYS_RAWIO process returns `-EPERM`.
- [ ] AC-9: Fuzzed maintenance ioctl offsets beyond `RIO_MAINT_SPACE_SZ` rejected with `-EINVAL`.

## Architecture

`Mport` lives in `drivers::rapidio::Mport`:

```
struct Mport {
  dev: KBox<Device>,                // class = rio_mport_class
  net: Mutex<Option<Arc<Net>>>,
  ops: &'static Ops,
  id: u8, index: u8,
  iores: Resource,
  inb_msg: [InbMsgRes; RIO_MAX_MBOX],
  outb_msg: [OutbMsgRes; RIO_MAX_MBOX],
  host_deviceid: u16,
  pwrite_lock: Mutex<()>,
  pwrites: Mutex<Vec<Arc<PortWriteHandler>>>,
  inb_resources: Mutex<Vec<Resource>>,
  outb_resources: Mutex<Vec<Resource>>,
  sys_size: u8,                     // 8 or 16 bit destid
  phys_efptr: u32, phys_rmap: u32,
  name: KString, priv: *mut c_void,
}

struct RioDev {
  dev: KBox<Device>,                // bus = rio_bus_type
  net: NonNull<Net>,
  vid: u16, did: u16, asm_vid: u16, asm_did: u16, asm_rev: u16,
  device_rev: u32, destid: u16, hopcount: u8,
  pef: u32, swpinfo: u32, comp_tag: u32,
  src_ops: u32, dst_ops: u32,
  rswitch: Option<NonNull<Switch>>,
  driver: AtomicPtr<RioDriver>,
  state: AtomicU32, use_count: Refcount,
}

struct Switch {
  rdev: NonNull<RioDev>,
  switchid: u32, port_ok: u32,
  add_entry / get_entry / clr_table / set_domain / get_domain / em_init / em_handle: Option<Fn>,
  nextdev: KBox<[Option<Arc<RioDev>>]>,     // per-port neighbor
  route_table: KBox<[u8; RIO_MAX_ROUTE_ENTRIES]>,
}
```

Mport register `rio_register_mport(port)`:
1. Assign `mport.id` from per-subsystem ida.
2. `device_register(&mport.dev)` with `class = &rio_mport_class`.
3. Initialize resource managers (inb/outb mbox, inb dbell, pwrite list).
4. `dev_set_drvdata` for the mport's host driver.
5. Optionally invoke `rio_init_mports` to drive scan.

Fabric scan `rio_disc_mport(mport)` / `rio_enum_mport(mport)` (in `rio-scan.c`):
1. Allocate `rio_net` (`rio_alloc_net`).
2. Read mport's own ID (`RIO_DID_CSR`), build root `rio_dev` for the mport.
3. Walk hopcount-based maintenance reads: per-port discover switch / endpoint; program switch route table to forward destid-X traffic to port-Y on the path.
4. For each discovered endpoint: `rio_add_device(rdev)` which `device_register`s on `rio_bus_type`; matching driver gets `probe(rdev)`.

Doorbell receive: hardware mport IRQ → mport driver calls `rio_in_dbell` (callback registered via `rio_request_inb_dbell`) with `(mport, src_id, dst_id, info)`. Subsystem looks up per-range registered callback and invokes.

Outbound DMA region: `rio_map_outb_region(mport, destid, rbase, size, rflags, &laddr)` programs the mport's outbound translation aperture so that local PCIe writes to `laddr` become SRIO NWRITEs to `destid` at `rbase`.

`/dev/rio_mport_cdev` ioctl `mport_cdev_ioctl`:
1. `mport_cdev_priv` derived from `filp->private_data`.
2. Dispatch:
   - `RIO_MPORT_MAINT_READ_LOCAL` / `_WRITE_LOCAL` — local mport config R/W via `rio_ops.lcread`/`_lcwrite`.
   - `RIO_MPORT_MAINT_READ_REMOTE` / `_WRITE_REMOTE` — destid+hop maint R/W via `rio_ops.cread`/`_cwrite`.
   - `RIO_MPORT_MAINT_HDID_SET` / `_COMPTAG_SET` — set local destid / comptag.
   - `RIO_DEV_ADD` / `_DEL` — userspace adds rio_dev (allows userspace-driven enumeration).
   - `RIO_MPORT_TRANSFER` / `_WAIT_FOR_ASYNC` — initiate DMA transfer, async completion.
   - `RIO_OBW_MAP_REQUEST` / `_OBW_FREE_MAP` — request outbound window mapping for userspace mmap.
   - `RIO_IBW_MAP_REQUEST` / `_IBW_FREE_MAP` — request inbound window over user pages.
   - `RIO_EVENT_RD` / `_WR` — read/write per-fd event ring (doorbells, port-writes).
3. Each ioctl validates per-struct user copy via `copy_from_user` with PAX_USERCOPY-bounded buffer; per-buffer size capped (`RIO_MAINT_SPACE_SZ` for maint, per-channel max for DMA).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mport_resource_oob` | OOB | per-mport `inb_msg[mbox]` / `outb_msg[mbox]` access bounded by `RIO_MAX_MBOX`. |
| `dbell_range_no_overlap` | OVERLAP | inbound doorbell range allocator refuses overlapping ranges. |
| `maint_offset_oob` | OOB | rio_mport_cdev maint ioctls reject offset+size past `RIO_MAINT_SPACE_SZ`. |
| `route_table_oob` | OOB | per-switch routing table index < `RIO_MAX_ROUTE_ENTRIES`. |
| `rdev_refcount_no_uaf` | UAF | `Arc<RioDev>` outlives switch nextdev pointers; del waits for last reference. |

### Layer 2: TLA+

`models/rapidio/dbell_dispatch.tla` — proves per-mport doorbell callback dispatch with concurrent register/unregister produces no UAF and never drops a registered-range delivery.

`models/rapidio/scan_topology.tla` — proves the depth-first scan with hop count and routing-table-programming terminates on any finite topology and visits every endpoint exactly once.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `rio_request_inb_dbell` post: range registered in callback table iff no overlap; release removes exact range | `Mport::request_inb_dbell`/`_release_inb_dbell` |
| `rio_map_outb_region` post: returns `laddr` such that writes to `[laddr, laddr+size)` produce NWRITE to `destid` at `[rbase, rbase+size)` | `Mport::map_outb_region` |
| `rio_add_device` / `rio_del_device` symmetry: every added rdev removed on mport unregister | `RioDev::add`/`_del` |
| `rio_mport_cdev` maint R/W: `count <= RIO_MAINT_SPACE_SZ - off` | `MportCdev::maint_*` |

### Layer 4: Verus functional

Per-mport messaging engine `add_outb_message` → peer `inb_msg_cb` invocation preserves message ordering within a mailbox (FIFO) and never delivers a half-message. Encoded as Verus refinement over the per-mbox queue.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

rapidio-core specific reinforcement:

- **`RIO_MAINT_SPACE_SZ` clamp** — maint R/W ioctls + sysfs `config` binfile clamp offset+size at 16 MiB.
- **Per-mport resource lists mutex-protected** — defense against concurrent mbox/dbell alloc race.
- **Per-switch route-table set/get serialized via switch ops** — defense against concurrent route-table corruption during scan.
- **Endpoint `use_count` refcount** — `rio_dev_get`/`_put` ensures rdev lives across in-flight ops.
- **Per-mport pwrite list under `pwrite_lock`** — defense against del-during-deliver race.
- **`rio_match_device` strict id-table match** — defense against off-table driver binding.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `rio_mport`, `rio_dev`, `rio_net`, `rio_switch`, per-mport resource records, per-fd `mport_cdev_priv`, and event-ring entries; ioctl payloads strictly size-bounded before `copy_from_user`/`copy_to_user`.
- **PAX_KERNEXEC** — RapidIO core text + `rio_bus_type`, per-mport `rio_ops`, per-switch route ops, per-driver `id_table`, and rio_mport_cdev fops in W^X kernel text; `__ro_after_init` for `rio_mport_class` and the switch-spec callback tables.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `rio_register_mport`, `rio_add_device`, `rio_disc_mport`, doorbell/pwrite callbacks, rio_mport_cdev ioctl dispatch, and DMA transfer completion paths.
- **PAX_REFCOUNT** — saturating `refcount_t` on `rio_dev.use_count`, per-fd `mport_cdev_priv.refcnt`, per-DMA-req `dma_ref`, and per-CM-channel state; overflow trap defeats hot-remove-vs-ioctl race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-mport resource records on release, route-table copies on switch teardown, DMA-bounce buffers on transfer free, and per-fd event rings so stale fabric metadata cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every rio_mport_cdev / rio_cm ioctl entry; user-supplied destid / hopcount / offset validated before deref.
- **PAX_RAP / kCFI** — per-mport `rio_ops` (`lcread`, `lcwrite`, `cread`, `cwrite`, `dsend`, `pwenable`, `open_inb_mbox`, `close_inb_mbox`, `add_outb_message`, `add_inb_buffer`, `map_inb`, `unmap_inb`), per-switch route callbacks, per-driver `probe`/`remove`, and rio_mport_cdev fops marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-mport MMIO / Tsi721 BAR pointer disclosure behind CAP_SYSLOG; suppress `%p` in scan-progress and pwrite-trace tracepoints.
- **GRKERNSEC_DMESG** — restrict scan-fail, route-program-fail, message-engine-err, and pwrite-overflow banners to CAP_SYSLOG so attackers cannot fingerprint fabric topology via dmesg.
- **mport ops kCFI** — per-mport `rio_ops` op-vector kCFI-typed; refuse mport register without correct ops type id.
- **Switch route-table validated** — per-switch `route_table` writes validated against `RIO_MAX_ROUTE_ENTRIES(sys_size)`; refuse out-of-range port-or-destid programming.
- **Doorbell PAX_USERCOPY** — `RIO_EVENT_RD` / `_WR` payloads size-validated; per-fd event ring bounded.
- **Message engine REFCOUNT** — per-outbound-message and per-inbound-buffer references via `refcount_t`; defense against UAF on hot-remove mid-transfer.
- **`/dev/rio_mport_cdev*` CAP_SYS_RAWIO** — opening the cdev requires CAP_SYS_RAWIO in the user namespace; refuse open under lockdown integrity tier (the cdev exposes raw fabric maintenance read/write).
- **Per-mport pwrite handler list locked** — del-during-deliver race avoided via `pwrite_lock` + RCU-like grace.
- **DMA-channel allocation per-fd** — `get_dma_channel` / `put_dma_channel` paired; defense against unbounded DMA-channel hold by a single fd.

Rationale: RapidIO exposes fabric-wide maintenance R/W, switch route-table programming, and zero-copy DMA into arbitrary destids — all from `/dev/rio_mport_cdevN`. A buggy ioctl, a missing route-table bound, or a UAF on rdev hot-remove can be turned into fabric-wide pwn (write to peer endpoint, reprogram switch route, DMA into arbitrary host memory via inbound window). CAP_SYS_RAWIO on the cdev, signed maintenance offset bounds, RAP/kCFI on mport ops + switch route callbacks, refcount-overflow trapping on rdev/transfer state, route-table validation, and sanitised release of resource records turn RapidIO from "open the node, own the fabric" into a structurally bounded fabric interconnect.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-mport host drivers (`tsi721.c`, `fsl_rio.c`) and per-switch internals (`switches/`) — each future Tier-3.
- `rio_cm.c` connection-mgmt details — future Tier-3 (`rio-cm.md`).
- RapidIO physical-layer (LP-Serial / LP-EP) — vendor IP, out of scope.
- 32-bit-only paths.
- Implementation code.
