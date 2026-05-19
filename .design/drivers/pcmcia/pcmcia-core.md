# Tier-3: drivers/pcmcia/{cs,ds,pcmcia_resource}.c — PCMCIA/CardBus core (socket services + CIS parser + resource arbiter)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/pcmcia/cs.c
  - drivers/pcmcia/cs_internal.h
  - drivers/pcmcia/ds.c
  - drivers/pcmcia/pcmcia_resource.c
  - drivers/pcmcia/cistpl.c
  - drivers/pcmcia/pcmcia_cis.c
  - drivers/pcmcia/rsrc_mgr.c
  - drivers/pcmcia/rsrc_nonstatic.c
  - drivers/pcmcia/socket_sysfs.c
  - include/pcmcia/ss.h
  - include/pcmcia/ds.h
  - include/pcmcia/cistpl.h
-->

## Summary

The PCMCIA/CardBus core implements the host side of the PC Card 16-bit and CardBus 32-bit specifications: a `pcmcia_socket` abstraction that any host-bridge driver (yenta, pd6729, soc_common, sa11xx, pxa2xx, etc.) registers against, the `pcmcia_bus_type` device-model integration via `ds.c`, the CIS (Card Information Structure) tuple parser that decodes the on-card metadata blob, and the I/O / memory / IRQ resource arbiter that hands per-card BARs to client drivers. CardBus cards are fundamentally PCI devices and are passed through to the PCI core via `cardbus.c`; this Tier-3 covers only the 16-bit PCMCIA core plus the socket-services layer shared by both modes.

Sources: `cs.c` (~902 lines, socket lifecycle + thread + event dispatch), `ds.c` (~1452 lines, `pcmcia_bus_type`, driver match, hotplug uevents), `pcmcia_resource.c` (~955 lines, per-function I/O window / mem window / IRQ alloc), `cistpl.c` (~1610 lines, CIS tuple walker), `pcmcia_cis.c` (CIS query helpers), `rsrc_mgr.c` + `rsrc_nonstatic.c` (host-bridge-supplied resource pools).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pcmcia_socket` (`include/pcmcia/ss.h`) | per-socket control block | `drivers::pcmcia::Socket` |
| `struct pcmcia_device` (`include/pcmcia/ds.h`) | per-function device on the pcmcia bus | `drivers::pcmcia::PcmciaDevice` |
| `struct pcmcia_driver` (`include/pcmcia/ds.h`) | client driver registration | `drivers::pcmcia::PcmciaDriver` |
| `struct pccard_operations` | host-bridge callbacks (init / get_status / set_socket / set_io_map / set_mem_map) | `drivers::pcmcia::SocketOps` |
| `pcmcia_register_socket(socket)` / `pcmcia_unregister_socket(socket)` | host-bridge socket registration | `Socket::register` / `_unregister` |
| `pcmcia_parse_events(socket, events)` / `pcmcia_parse_uevents(...)` | host-IRQ event dispatch into socket thread | `Socket::parse_events` / `_parse_uevents` |
| `pcmcia_reset_card(socket)` | hot-reset sequence (VS_SENSE -> reset -> PWR_UP -> CIS rescan) | `Socket::reset_card` |
| `pccard_register_pcmcia(socket, callback)` | wire socket into pcmcia bus | `Socket::register_bus` |
| `pcmcia_register_driver(driver)` / `pcmcia_unregister_driver(driver)` | client driver registration | `PcmciaDriver::register` / `_unregister` |
| `pcmcia_dev_present(dev)` | guarded "is card still present" check | `PcmciaDevice::is_present` |
| `pcmcia_request_window(dev, win, speed)` / `pcmcia_release_window(dev, win)` | memory-window alloc | `PcmciaDevice::request_window` |
| `pcmcia_request_io(dev)` / `pcmcia_release_io(dev)` | I/O-port alloc from `dev->resource[0..1]` | `PcmciaDevice::request_io` |
| `pcmcia_request_irq(dev, handler)` / `pcmcia_setup_irq(dev)` | per-card IRQ alloc + share | `PcmciaDevice::request_irq` |
| `pcmcia_enable_device(dev)` / `pcmcia_disable_device(dev)` | per-function configure (write CCR) | `PcmciaDevice::enable` / `_disable` |
| `pccard_read_tuple(socket, function, code, parse)` | CIS tuple lookup wrapper | `Socket::read_tuple` |
| `pccard_loop_tuple(socket, function, code, parse, priv, loop_tuple)` | iterate tuples of one code | `Socket::loop_tuple` |
| `pccard_validate_cis(socket, count)` | structural validation + count tuples | `Socket::validate_cis` |
| `pcmcia_replace_cis(socket, data, len)` | sysfs override of broken on-card CIS | `Socket::replace_cis` |
| `pcmcia_validate_mem(socket)` | confirm mem-window pool intersects RAM gap | `Socket::validate_mem` |
| `pcmcia_socket_class` (sysfs class) | `/sys/class/pcmcia_socket/socket%u/` | `Subsystem::class` |

## Compatibility contract

REQ-1: Socket abstraction matches `pccard_operations`: host bridges (yenta_socket / pd6729 / soc_common / pxa2xx_base / sa11xx_base / db1xxx_ss / electra_cf / omap_cf / bcm63xx_pcmcia) implement `init`/`get_status`/`set_socket`/`set_io_map`/`set_mem_map`/`suspend`; the core supplies the per-socket thread and event dispatch.

REQ-2: Per-socket kernel thread (`pccardd`) drives a state machine over `SS_DETECT` / `SS_READY` / `SS_BATDEAD` / `SS_BATWARN` / `SS_WRPROT` / `SS_3VCARD` / `SS_XVCARD` status bits and emits uevents for `add`/`remove`/`request`.

REQ-3: CIS tuple parser supports the full PC Card 8.0 tuple set — `CISTPL_DEVICE`, `CISTPL_VERS_1`, `CISTPL_MANFID`, `CISTPL_FUNCID`, `CISTPL_FUNCE`, `CISTPL_CONFIG`, `CISTPL_CFTABLE_ENTRY`, `CISTPL_LINKTARGET`, multi-function `CISTPL_LONGLINK_MFC`, end-of-chain `CISTPL_END` — and refuses unterminated or self-referential chains.

REQ-4: Per-function I/O window granularity: 1-byte / 2-byte / 4-byte windows mapped via `pccard_operations.set_io_map(socket, &io_map)`; up to `MAX_IO_WIN` simultaneous windows per socket.

REQ-5: Per-function memory window granularity: per-host-bridge `MIN_PAGE` (typically 4 KB) alignment; per-window `attribute`/`common` selector chooses between attribute memory (CIS + CCR) and common memory.

REQ-6: IRQ allocation: per-card IRQ routed through host-bridge ISA-IRQ remap (yenta uses PCI IRQ for CardBus; 16-bit PCMCIA shares the host-bridge IRQ via `IRQF_SHARED`).

REQ-7: Resource pool population: `rsrc_nonstatic` exposes `/sys/class/pcmcia_socket/socket%u/available_resources_io` / `_mem` so platform code can hand the host bridge a constrained physical range, and the arbiter refuses overlap with `iomem_resource` busy regions.

REQ-8: Hotplug uevents: `add` / `remove` / `change` propagated via `kobject_uevent_env` with `PCMCIA_MANFID`/`PCMCIA_PROD_ID`/`PCMCIA_FUNCID` env so udev can match.

REQ-9: 16-bit (`pcmcia_bus_type`) and 32-bit CardBus (`pci_bus_type` via `cardbus.c`) coexist on the same socket: socket-services owns Vcc/Vpp/reset; CardBus cards are then enumerated as PCI children.

REQ-10: Suspend/resume: per-socket `pccard_operations.suspend` powers card down on system sleep; on resume the CIS is re-read and the per-function configuration is re-applied (no client-driver state assumed across S3).

REQ-11: `pcmcia_replace_cis(socket, data, len)` (sysfs `cis` write) accepts an admin-supplied CIS blob for broken cards; the override is bounded to `CISTPL_MAX_CIS_SIZE` (0x200) and stored per-socket.

## Acceptance Criteria

- [ ] AC-1: Modprobe `yenta_socket` on a laptop with a Cardbus bridge enumerates `/sys/class/pcmcia_socket/socket0/`.
- [ ] AC-2: Insert a 16-bit PCMCIA NIC (e.g. 3c589) — udev sees `add` event with correct `PCMCIA_MANFID`/`PCMCIA_PROD_ID`; client driver binds; eth interface appears.
- [ ] AC-3: Eject card while client driver bound — `remove` uevent fires, client `.remove` callback runs, socket returns to empty state.
- [ ] AC-4: CardBus 32-bit NIC (e.g. 3c575) enumerates under `pci_bus_type` with BARs assigned out of the host bridge's CardBus resource window.
- [ ] AC-5: Suspend/resume with card inserted: card re-detected post-resume, CIS re-read, client driver re-bound.
- [ ] AC-6: Malformed CIS (truncated chain) is rejected by `pccard_validate_cis` with a `dmesg` diagnostic; socket remains in `DETECT_PENDING` until ejected.
- [ ] AC-7: `pcmcia_replace_cis` sysfs override accepted from CAP_SYS_ADMIN process; client binding succeeds against override.
- [ ] AC-8: `pcmciautils` (`pccardctl`) commands `ident`, `eject`, `insert`, `suspend`, `resume` all succeed.

## Architecture

`Socket` is the per-physical-socket runtime, registered by each host-bridge driver:

```
struct Socket {
  ops: &'static SocketOps,
  features: SocketFeatures,            // SS_CAP_PCCARD | SS_CAP_CARDBUS | SS_CAP_MEM_ALIGN | ...
  state: AtomicU32,                    // SOCKET_PRESENT | SOCKET_INUSE | SOCKET_SUSPEND
  socket_no: u8,
  thread: Option<KThread>,             // pccardd/N
  thread_wait: WaitQueue,
  thread_events: AtomicU32,            // SS_DETECT | SS_BATDEAD | SS_BATWARN | SS_READY | ...
  fake_cis: Mutex<Option<KVec<u8>>>,   // sysfs override
  cis_cache: Mutex<CisCache>,
  pcmcia_pfc: u8,                      // pseudo-multi-function-card hack flag
  pcmcia_state: PcmciaState,           // dead | empty | suspended | active
  cb_dev: Option<Arc<PciDev>>,         // CardBus child PCI device
  resource_ops: &'static ResourceOps,  // rsrc_nonstatic vs. rsrc_iodyn
  resource_data: NonNull<()>,
  parent: Arc<Device>,
}

struct PcmciaDevice {
  socket: Arc<Socket>,
  func: u8,                            // multi-function card index
  function_config: Arc<ConfigT>,       // shared across functions
  dev: Device,                         // device-model node on pcmcia_bus_type
  resource: [Resource; PCMCIA_NUM_RESOURCES],
  manf_id: u16,
  card_id: u16,
  prod_id: [Option<KString>; 4],
  has_manf_id: bool,
  has_card_id: bool,
  has_func_id: bool,
  func_id: u8,
  open: AtomicI32,
  allow_func_id_match: AtomicBool,
}
```

Socket registration `Socket::register`:
1. `socket.socket_no` allocated from `pcmcia_socket_no_ida`.
2. `device_register(&socket.dev)` under `pcmcia_socket_class`.
3. Spawn `pccardd/N` kernel thread.
4. `pccard_register_pcmcia(socket, &bus_callback)` wires the bus layer in.
5. `ops.init(socket)` then `ops.get_status(socket, &status)` — if SS_DETECT, post `SS_DETECT` event to thread.

Per-socket thread `pccardd` loop:
1. `wait_event_interruptible(thread_wait, thread_events != 0 || kthread_should_stop())`.
2. Read `thread_events` atomically + clear.
3. `SS_DETECT` raised: power up sequence — set Vcc to 5V or 3.3V (chosen from `SS_3VCARD`/`SS_XVCARD`), `RESET=1` for 10 ms, `RESET=0`, wait `SS_READY`.
4. Read raw CIS via attribute-memory window: `pccard_validate_cis(socket, &count)`; refuse on malformed.
5. `pcmcia_card_add(socket)` enumerates per-function `pcmcia_device` children + calls `device_add`; bus match drives client `.probe`.
6. `SS_DETECT` cleared (eject): `pcmcia_card_remove(socket, leftover)` runs each child's `.remove`, unrefs.

CIS parsing `Socket::read_tuple(function, code, parse)`:
1. Map attribute-memory window via `ops.set_mem_map(socket, &mem_map)` with `MAP_ATTRIB`.
2. Walk from offset 0: read 1-byte tuple code + 1-byte link; bounded by `CISTPL_MAX_CIS_SIZE` (0x200).
3. `CISTPL_LONGLINK_*` follows chain to common memory or to next function; chains are depth-limited.
4. On code match: copy at most `MAX_TUPLE_SIZE` (0xff) data bytes into a `cistpl_t` and call `parse(tuple, parsed)`.
5. Return -ENOSPC on overlong tuple, -EINVAL on truncated chain.

Per-function I/O allocation `PcmciaDevice::request_io`:
1. Validate `dev.resource[0].start`/`.end` against host-bridge IO pool via `resource_ops.find_io`.
2. `__request_region(&ioport_resource, base, len, name, 0)` — taints if collision.
3. `ops.set_io_map(socket, &io_map { map: 0, flags: MAP_ACTIVE | MAP_USE_WAIT, speed, start, stop })`.
4. Store window cookie in `dev.resource[0].flags |= IO_DATA_PATH_WIDTH_*`.

Per-function IRQ allocation `PcmciaDevice::request_irq`:
1. Determine IRQ: host-bridge supplied (`socket.pci_irq` for yenta) or legacy ISA mask probe.
2. Write CCR (Configuration Control Register) at attribute-memory offset `function_config.base + CISREG_COR` with `COR_LEVEL_REQ | function_index`.
3. `request_irq(irq, handler, IRQF_SHARED, dev_name, dev)`.

CardBus path `cardbus_add` (cross-ref `drivers/pci/`):
1. Once `SS_CB_CARD` reported, `pci_scan_bus_parented(socket.dev, socket.cb_dev_busn)` enumerates the CardBus child PCI bus.
2. PCI core assigns BARs out of host-bridge CardBus window; client driver binds via `pci_bus_type`.

Resource arbiter `rsrc_nonstatic`:
- Per-socket `socket_data.io_db` and `socket_data.mem_db` linked lists of available `[base,len]` ranges, configured via `/sys/class/pcmcia_socket/socket%u/available_resources_{io,mem}` write.
- `find_io(socket, attr, base, num, align, parent)` walks `io_db`, skips ranges that overlap `iomem_resource` busy children, returns first fit.
- `validate_mem(socket)` issues a memory probe (write/read pattern at candidate base) to confirm the host bridge actually decodes the range.

## Hardening

- All `pccard_operations` callbacks invoked only from the per-socket thread or from initialization context — no concurrent CIS reads from multiple CPUs against one socket.
- `cistpl.c` chain walker bounded by `CISTPL_MAX_CIS_SIZE` (0x200) total bytes and `MAX_TUPLES` chain depth; defense against malicious CIS triggering infinite link loop.
- `pcmcia_replace_cis(socket, data, len)` rejects `len > CISTPL_MAX_CIS_SIZE` and requires `CAP_SYS_ADMIN` via sysfs ownership.
- `find_io` / `find_mem` reject ranges overlapping `iomem_resource.busy` children (kernel RAM, PCI BARs, MMIO).
- IRQ alloc uses `IRQF_SHARED` and refuses non-shareable lines on shared host bridges.
- `pcmcia_dev_present(dev)` guard against UAF on ejected-card races: clients call this before touching `dev->resource[]`.
- Per-socket `pccardd` thread is the single writer of `pcmcia_state`; userspace and IRQ handlers only post events.
- `cb_dev` (CardBus PCI child) refcounted via `pci_dev_get`; teardown order: PCI remove → socket-services release.
- Suspend path forces `Vcc=0` then `RESET=1` so an inserted card cannot DMA across the host suspend.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `pcmcia_socket`, `pcmcia_device`, `cistpl_t`, and CIS-cache pages; sysfs `cis` write into `fake_cis` strictly bounded to `CISTPL_MAX_CIS_SIZE`.
- **PAX_KERNEXEC** — PCMCIA core (`cs.c`, `ds.c`) text W^X; `pccard_operations`, `pcmcia_driver` vtables, and `pcmcia_bus_type` ops placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel stack on `pccardd` thread entry, `pcmcia_bus_match`, CIS tuple walk, and resource-alloc syscall entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on `pcmcia_socket` (`socket.refcount`), `pcmcia_device.dev.kobj`, and CIS-cache buffers; overflow trap defeats hotplug-race UAF.
- **PAX_MEMORY_SANITIZE** — zero-on-free for CIS-cache pages, `fake_cis` buffer, and `pcmcia_device` slab on eject so prior-card metadata cannot leak into the next insertion.
- **PAX_UDEREF** — SMAP/PAN enforced on `pcmcia_replace_cis` sysfs write and `ioctl` paths under the deprecated `pcmcia_dev_ctrl` shim.
- **PAX_RAP / kCFI** — `pccard_operations`, `pcmcia_bus_type` callbacks, `pcmcia_driver.probe`/`.remove`, and CIS-parse function pointers marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms disclosure of host-bridge MMIO bases and `pcmcia_socket` pointers behind CAP_SYSLOG; `%p` neutered in PCMCIA tracepoints.
- **GRKERNSEC_DMESG** — restrict CIS-parse failure, malformed-tuple, and resource-overlap banners to CAP_SYSLOG.
- **CAP_SYS_RAWIO for socket services** — raw socket I/O ioctls (legacy `pcmcia_dev_ctrl`, attribute-memory window write via debugfs) require CAP_SYS_RAWIO in addition to file mode.
- **CAP_SYS_ADMIN on hotplug control** — `/sys/class/pcmcia_socket/socket%u/{card_eject,card_insert,resource_mem,resource_io,cis}` writes gated by CAP_SYS_ADMIN.
- **CIS tuple-parser bounded** — total CIS length capped at `CISTPL_MAX_CIS_SIZE`, per-tuple data at `MAX_TUPLE_SIZE`, chain depth bounded; defense against malicious CIS triggering kernel OOB read.
- **Resource-pool validation** — `available_resources_{io,mem}` writes refused when range overlaps `iomem_resource.busy` children or kernel image; defense against admin-mistyped pool exposing RAM to attribute-memory mapping.
- **Hotplug uevent sanitization** — `PCMCIA_PROD_ID*` env strings stripped of non-printable and shell metacharacters before passing to `kobject_uevent_env`.

Rationale: PCMCIA sockets accept attacker-controlled hardware on physical insertion, and the on-card CIS is the first thing the kernel parses. A malformed CIS, an unbounded tuple chain, or a resource-arbiter overlap silently elevates an inserted card from "decoded peripheral" into "kernel-memory peer". Bounded CIS parsing, refcount overflow trapping, kCFI on `pcmcia_driver` vtables, and CAP_SYS_ADMIN on resource-pool writes turn the socket from "trust whatever's plugged in" into a contained enrolment boundary; the suspend-forces-Vcc-0 rule keeps an inserted card from DMA-ing across system sleep.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-host-bridge drivers (yenta_socket, pd6729, soc_common, sa11xx_base, pxa2xx_base) — Tier-4 each.
- CardBus PCI enumeration (covered by `drivers/pci/` Tier-3).
- 32-bit-only or m68k-only platform PCMCIA bridges.
- Implementation code.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cis_walk_bounded` | OOB | total tuple-walk byte count <= `CISTPL_MAX_CIS_SIZE`; per-tuple data length <= `MAX_TUPLE_SIZE`. |
| `cis_chain_depth_bounded` | TERMINATION | `CISTPL_LONGLINK_*` chain depth bounded; cycle in chain rejected with -EINVAL. |
| `socket_state_no_uaf` | UAF | `Arc<Socket>` outlives all `PcmciaDevice` children; child remove drains before socket free. |
| `io_pool_no_overlap` | EXCLUSIVITY | `find_io` returned range is disjoint from every `iomem_resource.busy` child. |

### Layer 2: TLA+

`models/pcmcia/socket_thread.tla` (parent-declared): proves single-writer invariant on `pcmcia_state` between `pccardd`, the host-bridge IRQ event poster, and the sysfs `card_{eject,insert}` writer; no transition is dropped, no two add/remove pairs interleave.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Socket::reset_card` post: card is either in `DETECT_PENDING` (eject during reset) or fully readied; never half-configured | `Socket::reset_card` |
| `PcmciaDevice::request_io` post: every byte of returned region is owned by this device's `dev.resource[]` | `PcmciaDevice::request_io` |
| `pccard_validate_cis` post: returns OK only if every tuple's `code + 2 + len` <= remaining-bytes-from-start | `Socket::validate_cis` |
| Hotplug uevent invariant: every `add` for `(socket, function)` is followed by exactly one `remove` before another `add` | `pccardd` thread |

### Layer 4: Verus/Creusot functional

Card insert → host-bridge `SS_DETECT` event → `pccardd` runs reset+CIS+enumerate → `device_add` → udev `add` event; symmetric on eject. Encoded as a Verus state-machine refinement of the abstract `inserted`/`removed` events with no observable inconsistency between sysfs state and `pcmcia_bus_type` device list.
