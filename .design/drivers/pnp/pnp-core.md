# Tier-3: drivers/pnp/{core,manager,resource}.c — Plug-and-Play framework (PNPACPI + PNPBIOS + ISAPNP)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/pnp/core.c
  - drivers/pnp/card.c
  - drivers/pnp/driver.c
  - drivers/pnp/interface.c
  - drivers/pnp/manager.c
  - drivers/pnp/resource.c
  - drivers/pnp/support.c
  - drivers/pnp/system.c
  - drivers/pnp/quirks.c
  - drivers/pnp/base.h
  - drivers/pnp/pnpacpi/core.c
  - drivers/pnp/pnpacpi/rsparser.c
  - drivers/pnp/pnpacpi/pnpacpi.h
  - drivers/pnp/pnpbios/core.c
  - drivers/pnp/pnpbios/bioscalls.c
  - drivers/pnp/pnpbios/rsparser.c
  - drivers/pnp/pnpbios/proc.c
  - drivers/pnp/pnpbios/pnpbios.h
  - drivers/pnp/isapnp/
  - include/linux/pnp.h
  - include/linux/pnpbios.h
-->

## Summary

The `pnp` subsystem is the Linux unification layer for legacy plug-and-play discovery on x86: it presents `pnp_bus_type` with three protocols underneath — `pnpacpi` (the modern path, parses ACPI `_CRS`/`_PRS`/`_SRS` resource descriptors), `pnpbios` (the pre-ACPI Compaq/Microsoft PnP BIOS spec, calls into 16-bit real-mode firmware code), and `isapnp` (genuine ISA Plug-and-Play card protocol with the legacy CSN / serial-isolation handshake). All three protocols emit `struct pnp_dev` instances on the shared bus; per-device drivers (legacy `i8042`, `serial_pnp`, `parport`, `floppy`, sound card legacy stubs, etc.) match by EISA-ID via `pnp_device_id` and consume the per-device `resource[]` array assigned by the `manager.c` arbiter from the per-device `options` set advertised by the protocol's resource parser.

Sources: `core.c` (~224 lines, `pnp_bus_type`, protocol/device/card registration), `manager.c` (~429 lines, per-device resource assignment), `resource.c` (~755 lines, resource lookup/validation, possible/preferred config tables), `interface.c` (sysfs `id`, `options`, `resources` files), `pnpacpi/core.c` + `rsparser.c` (ACPI protocol + `_CRS`/`_PRS`/`_SRS` parse), `pnpbios/core.c` + `bioscalls.c` + `rsparser.c` (legacy BIOS protocol + 16-bit thunk), `isapnp/` (ISA PnP CSN/serial-isolation).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pnp_protocol` (`include/linux/pnp.h`) | per-protocol descriptor (PNPACPI / PNPBIOS / ISAPNP) | `drivers::pnp::Protocol` |
| `struct pnp_dev` | per-PnP device on bus | `drivers::pnp::PnpDev` |
| `struct pnp_card` | per-PnP card (1+ devices) | `drivers::pnp::PnpCard` |
| `struct pnp_driver` | client driver registration on `pnp_bus_type` | `drivers::pnp::PnpDriver` |
| `struct pnp_card_driver` | client driver for a multi-device card | `drivers::pnp::PnpCardDriver` |
| `struct pnp_resource` / `struct pnp_option` | per-device resource (assigned) and option (possible config) | `drivers::pnp::PnpResource` / `PnpOption` |
| `pnp_bus_type` | bus_type | `drivers::pnp::PnpBusType` |
| `pnp_register_protocol(protocol)` / `pnp_unregister_protocol(protocol)` (`core.c`) | per-protocol register | `Protocol::register` / `_unregister` |
| `pnp_add_device(dev)` / `__pnp_remove_device(dev)` | per-device register | `PnpDev::add` / `_remove` |
| `pnp_alloc_dev(protocol, id, pnpid)` / `pnp_free_resources(dev)` | allocator | `PnpDev::alloc` / `_free_resources` |
| `pnp_register_driver(driver)` / `pnp_unregister_driver(driver)` | client driver register | `PnpDriver::register` / `_unregister` |
| `pnp_get_resource(dev, type, bar)` (`resource.c`) | resource accessor | `PnpDev::get_resource` |
| `pnp_possible_config(dev, type, start, size)` / `pnp_range_reserved(start, end)` (`resource.c`) | resource arbiter helpers | `PnpDev::possible_config` / `pnp_range_reserved` |
| `pnp_activate_dev(dev)` / `pnp_disable_dev(dev)` / `pnp_start_dev(dev)` / `pnp_stop_dev(dev)` (`manager.c`) | per-device enable + start | `PnpDev::activate` / `_disable` / `_start` / `_stop` |
| `pnp_assign_resources(dev, depnum)` | run per-device resource assignment over `options[depnum]` | `PnpDev::assign_resources` |
| `pnp_add_id(dev, id)` / `pnp_add_card_id(card, id)` | extra-EISA-ID tagging | `PnpDev::add_id` / `PnpCard::add_id` |
| `struct pnp_protocol pnpacpi_protocol` (`pnpacpi/core.c`) | ACPI protocol instance | `pnpacpi::Protocol` |
| `pnpacpi_parse_resource_option_data(dev)` / `pnpacpi_parse_allocated_resource(dev)` / `pnpacpi_encode_resources(dev, &buffer)` (`pnpacpi/rsparser.c`) | parse `_PRS`/`_CRS` + encode `_SRS` | `pnpacpi::Parser::*` |
| `struct pnp_protocol pnpbios_protocol` (`pnpbios/core.c`) | PnP BIOS protocol instance | `pnpbios::Protocol` |
| `pnp_bios_dev_node_info(&data)` / `pnp_bios_get_dev_node(&num, &boot, &data)` / `pnp_bios_set_dev_node(num, boot, &data)` (`pnpbios/bioscalls.c`) | 16-bit thunked BIOS calls | `pnpbios::BiosCall::*` |
| `pnpbios_parse_data_stream(dev, p)` / `pnpbios_encode_allocated_resource(dev, p, end)` (`pnpbios/rsparser.c`) | parse + encode 1-byte/2-byte tag stream | `pnpbios::Parser::*` |
| `pnp_platform_devices` (`core.c` global) | exposed marker that platform PnP-style discovery has run | `Subsystem::platform_devices` |
| `isapnp_card_list` / `isapnp_proto` (`isapnp/`) | ISAPNP protocol instance | `isapnp::Protocol` |

## Compatibility contract

REQ-1: `pnp_bus_type` exposes per-device `id` (primary EISA-ID), `compatible_ids[]`, `resources[]` (PnP arbiter-assigned), `options[]` (possible config alternatives) on sysfs at `/sys/bus/pnp/devices/%s/`.

REQ-2: Per-device-id match: `pnp_device_id.id` is a 7-character EISA-format ID (e.g. `PNP0303` for AT keyboard, `PNP0501` for 16550 UART); `pnp_bus.match` walks driver's `id_table` against `dev.id` + `dev.compatible_ids[]`.

REQ-3: Resource model: every `pnp_dev` carries a `resources[]` list of `pnp_resource` entries (one per IO/MEM/IRQ/DMA range) + an `options[]` list of `pnp_option` describing each possible-config alternative; `manager.c` chooses one alternative at activation.

REQ-4: PNPACPI protocol: discovers PnP devices as ACPI namespace nodes with non-`PNP0A03` (root bridge) `_HID`/`_CID`; parses `_PRS` (possible) / `_CRS` (current) / `_SRS` (set), encoding via ACPICA's resource-templates; `_DIS` to disable, `_PSx` for power state. Always preferred over PNPBIOS when ACPI is enabled.

REQ-5: PNPBIOS protocol: 16-bit real-mode firmware interface (PnP BIOS 1.0a spec, Compaq/Microsoft 1994); kernel sets up `pnp_bios_callpoint` thunk to invoke 16-bit code from 32-bit protected mode. Only active when CONFIG_PNPBIOS=y, no ACPI present, and on i386 only (no 64-bit equivalent). Modern systems lockdown this protocol.

REQ-6: ISAPNP protocol: physical ISA-bus serial-isolation protocol (Plug-and-Play ISA spec 1.0a); per-card CSN (Card Select Number) issued via wake-key, serial-EEPROM resource map read out via I/O port pair `0x279`/`0xa79`/`0x203`. Per-IRQ/DMA/IO assignment programmed back via per-CSN reg writes. Becomes empty on systems with no real ISA bus (almost everyone).

REQ-7: Resource arbiter (`manager.c`): per-device `pnp_assign_resources(dev, depnum)` walks `dev.options[depnum]` for IRQ/DMA/IO/MEM alternatives, picks first that survives validation against `pnp_range_reserved` (refuses overlap with kernel image, reserved BIOS regions, in-use `iomem_resource`/`ioport_resource` busy children).

REQ-8: `pnp_activate_dev(dev)`: calls protocol's `set` op (PNPACPI: `_SRS`; PNPBIOS: `pnp_bios_set_dev_node`; ISAPNP: per-CSN register writes); then protocol's `enable` op to set ACPI `_PSx` / PnPBIOS activate / ISAPNP I/O-range-check-bit.

REQ-9: `pnp_disable_dev(dev)`: protocol's `disable` op (`_DIS` / PnpBios deactivate / ISAPNP deactivate). Per-device refcount tracked; refuses disable when in-use.

REQ-10: Quirks (`quirks.c`): per-EISA-ID workarounds (e.g. SMC FDC37C669 needs explicit IRQ list, IBM ThinkPad BIOS reports wrong serial-port irq).

REQ-11: `pnp_card` groups N `pnp_dev` from one physical card (typical for ISAPNP sound cards); per-card driver receives the bundle and probes each device individually.

REQ-12: PNPBIOS LOCKDOWN: pnpbios calls execute 16-bit firmware in real mode with kernel privilege; on lockdown-integrity kernels the entire PNPBIOS path is unconditionally disabled, and runtime probing requires the kernel command line to not have set lockdown.

## Acceptance Criteria

- [ ] AC-1: On a modern ACPI x86 system, `pnp_platform_devices == true` and `/sys/bus/pnp/devices/` lists at least the `PNP0303` keyboard and `PNP0F03` mouse aliases.
- [ ] AC-2: `/sys/bus/pnp/devices/00:0a/options` lists IRQ/IO ranges parsed from ACPI `_PRS` for the device.
- [ ] AC-3: Writing `disable` to `/sys/bus/pnp/devices/%s/resources` while no driver bound succeeds; ACPI `_DIS` runs.
- [ ] AC-4: `serial_pnp` driver matches `PNP0501` 16550 UART → enumerates the legacy COM port as `ttyS0` with the IRQ/IO assigned by ACPI.
- [ ] AC-5: ISAPNP test on a board with a CMI8330 ISA sound card: `cat /proc/isapnp` lists CSNs + resource maps.
- [ ] AC-6: PNPBIOS path: on a pre-ACPI machine with CONFIG_PNPBIOS=y, `/proc/bus/pnp/00/` lists BIOS-reported device nodes; on a LOCKDOWN_INTEGRITY_MAX kernel the same machine refuses to enable PNPBIOS.
- [ ] AC-7: `pnp_possible_config(dev, IORESOURCE_IO, 0x3f8, 8)` returns true for a `PNP0501` device's option set including `0x3f8`.
- [ ] AC-8: Resource arbiter rejects assignment that would overlap with kernel `_text` or with existing `iomem_resource.busy` child.

## Architecture

`PnpDev` is the per-device runtime, owned by a protocol:

```
struct PnpDev {
  dev: Device, protocol: Arc<Protocol>, number: u8, status: PnpStatus,  // ATTACHED | DETACHED
  card: Option<Arc<PnpCard>>, driver: Option<Arc<PnpDriver>>,
  data: NonNull<()>,                    // protocol-private (ACPI handle / BIOS node # / ISAPNP CSN)
  res: Mutex<PnpResources>,             // assigned resources + option alternatives
  id: [u8; 8], active: AtomicBool, capabilities: PnpCaps,
  procdir: Option<ProcDirEntry>,        // /proc/bus/isapnp/00/ for legacy paths
}

struct PnpResources { resource: Vec<PnpResource>, option: Vec<PnpOption>, dep_options: Vec<PnpOption> }
struct PnpResource { type_: ResourceType, range: Range, flags: u32 }  // IO | MEM | IRQ | DMA | BUS
struct PnpOption { type_: ResourceType, priority: u8, spec: PnpOptionSpec }  // (start,end,align,size) or mask

struct Protocol {
  name: KString, number: u8,
  get: fn(dev) -> Result<()>,           // refresh dev.resource[] from HW state
  set: fn(dev) -> Result<()>,           // push dev.resource[] to HW
  disable: fn(dev) -> Result<()>,
  can_wakeup: fn(dev) -> bool, suspend: fn(dev, state) -> Result<()>, resume: fn(dev) -> Result<()>,
  devices: Mutex<List<Arc<PnpDev>>>, cards: Mutex<List<Arc<PnpCard>>>,
}
```

Protocol registration `Protocol::register`:
1. Append to `pnp_protocols` list under `pnp_lock`.
2. Trigger protocol's enumeration (PNPACPI: walk ACPI namespace; PNPBIOS: call `pnp_bios_dev_node_info` + iterate; ISAPNP: serial-isolation rounds).
3. For each discovered device, `PnpDev::alloc` + `PnpDev::add`.

PNPACPI enumeration:
1. `acpi_get_devices` walks ACPI namespace for `_HID`/`_CID`.
2. Skip devices whose `_HID` is in `excluded_id_list` (PCI bridges, etc.).
3. For matching node: `pnp_alloc_dev(&pnpacpi_protocol, id, pnpid)`; stash `acpi_handle` in `dev.data`.
4. `pnpacpi_parse_allocated_resource(dev)` runs ACPI `_CRS` → fills `dev.res.resource[]`.
5. `pnpacpi_parse_resource_option_data(dev)` runs ACPI `_PRS` → fills `dev.res.option[]`.
6. `pnp_add_device(dev)` — registers on bus.

PNPBIOS enumeration:
1. `pnp_bios_dev_node_info` returns node count + buffer size.
2. For each node 0..N: `pnp_bios_get_dev_node(&num, BOOT_CONFIG, &data)` returns the per-device tagged-resource stream.
3. `pnpbios_parse_data_stream(dev, p)` walks the 1-byte/2-byte tags (IRQ format, DMA format, fixed-IO, range-IO, fixed-MEM, range-MEM, dependent function start/end, end tag).
4. `pnp_add_device(dev)`.

ISAPNP enumeration:
1. Send LFSR wake-key sequence to port `0x279` (ADDRESS) + `0xa79` (WRITE_DATA).
2. Serial-isolation loop reads bit-by-bit each card's 64-bit serial-EEPROM ID via port `0x203` (READ_DATA).
3. Per-card: assign CSN, walk per-CSN resource map.
4. `pnp_alloc_dev` + `pnp_add_device` per logical device.

Resource arbiter `pnp_assign_resources(dev, depnum)`:
1. Load `dev.res.option` for the chosen dependent-function index.
2. For each option (IRQ/DMA/IO/MEM):
   - Walk the alternative list.
   - For each candidate: validate via `pnp_check_port`/`_irq`/`_mem`/`_dma` against `ioport_resource`/`iomem_resource`/`pnp_unassigned_irq_resource_list` and per-quirk constraints.
   - First passing alternative wins; record in `dev.res.resource[]`.
3. Return -EBUSY if no alternative survives.

Activation `pnp_activate_dev`:
1. `dev.protocol.set(dev)` pushes per-resource assignment back to firmware (PNPACPI: build `_SRS` resource template; PNPBIOS: `pnp_bios_set_dev_node`; ISAPNP: per-CSN IRQ/IO/MEM reg writes).
2. Per-protocol "enable" op turns on device (`_PSx D0`, PnPBIOS activate, ISAPNP I/O-range-check-bit clear).
3. `dev.active = true`.

PNPBIOS thunk (`bioscalls.c`):
1. Kernel allocates a 4 KB scratch area in low memory.
2. Switches to PNPBIOS code segment + data segment + 16-bit stack via long jump through `pnp_bios_callpoint`.
3. 16-bit BIOS code executes, returns via `lretl`.
4. Kernel switches back to 32-bit mode, validates return code.
5. This crosses W^X (BIOS code is real-mode), CFI (no kCFI control), and PaX boundaries — fatal on lockdown-max.

sysfs interface (`interface.c`):
- `/sys/bus/pnp/devices/%s/id` — primary EISA-ID.
- `/sys/bus/pnp/devices/%s/options` — list of per-resource-type alternatives, parsed from `dev.res.option`.
- `/sys/bus/pnp/devices/%s/resources` — current assignment; writable for `disable`/`activate`/`io`/`irq`/`mem`/`dma` commands (CAP_SYS_ADMIN gated).

## Hardening

- Resource arbiter rejects assignment overlapping kernel `_text`/`_rodata`/`_data`/`_bss` and active `iomem_resource.busy` children.
- `pnp_check_port` enforces 16-bit IO-port wrap-around safety.
- `pnp_check_irq` validates against `nr_irqs` and refuses IRQ already claimed non-shareable.
- Per-protocol `set` op idempotent: calling twice with same `resource[]` is no-op.
- PNPBIOS thunk runs with IRQs disabled and a per-call timeout watchdog; on timeout the thunk is considered fatal and PNPBIOS is unregistered.
- ISAPNP serial-isolation timing constants are bounded; refuse infinite retry on isolation failure.
- sysfs `resources` write command parser is bounded by line length + token count; refuses malformed syntax.
- Per-`pnp_dev` `active` flag enforces single-activation; double-activation refused.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `pnp_dev`, `pnp_card`, `pnp_resource`, `pnp_option`, and per-protocol private payloads; sysfs `resources` write bounded.
- **PAX_KERNEXEC** — pnp core (`core.c`, `manager.c`, `resource.c`, `interface.c`, `support.c`, `driver.c`) and per-protocol cores (`pnpacpi/*`, `pnpbios/*` non-thunk, `isapnp/*`) in W^X kernel text; `pnp_bus_type` ops, `pnp_protocol` callbacks, `pnp_driver.{probe,remove,suspend,resume}` vtables placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel stack on `pnp_bus_match`, `pnp_assign_resources`, `pnpacpi_parse_allocated_resource`, ISAPNP serial-isolation routines, and sysfs `resources` write handler.
- **PAX_REFCOUNT** — saturating `refcount_t` on `pnp_dev`, `pnp_card`, and `pnp_protocol.devices` list refs; overflow trap defeats hotplug-race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `pnp_dev`, `pnp_resource`, `pnp_option`, ACPI resource-buffer copies, and PNPBIOS scratch areas so stale resource layouts cannot bleed across device re-probe.
- **PAX_UDEREF** — SMAP/PAN enforced on `/sys/bus/pnp/devices/%s/resources` write, `/proc/bus/pnp/` write, and `pnpbios` userland thunk entry; userspace pointer deref strictly bounded.
- **PAX_RAP / kCFI** — `pnp_protocol` callbacks (`get`/`set`/`disable`/`suspend`/`resume`), `pnp_driver` vtables, and ACPI resource-template-encode/decode helpers marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms disclosure of ACPI handles, PNPBIOS callpoint, and `pnp_dev` pointers behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict resource-overlap, ACPI `_SRS`-fail, PNPBIOS-thunk-fail, and ISAPNP-isolation-fail banners to CAP_SYSLOG.
- **Resource-arbiter range bounds** — every `pnp_check_{port,mem,irq,dma}` validates candidate against kernel image (`_text`..`_end`), `iomem_resource.busy` children, `ioport_resource.busy` children, and per-arch reserved BIOS regions; refuse any overlap.
- **PNPACPI `_CRS` parse** — ACPICA resource-walker callbacks validated for descriptor-length + producer-resource-template correctness; refuse malformed `_CRS`/`_PRS` blob; bounded recursion.
- **PNPBIOS lockdown gate** — the entire PNPBIOS protocol (16-bit real-mode thunk) is unconditionally disabled when `kernel_locked_down(LOCKDOWN_PNPBIOS)` returns true; cannot be re-enabled at runtime; on non-lockdown kernels every `bioscalls.c` entry additionally requires CAP_SYS_RAWIO.
- **Real-mode thunk discipline** — PNPBIOS thunk page is W^X-violating by design (real-mode code execution); placed in a dedicated thunk-only mapping, IRQs disabled across call, per-call watchdog forces unregister on timeout; refuse thunk re-entry.
- **ISAPNP CSN bounds** — Card Select Number space limited to 0..PNP_MAX_CSN; serial-isolation rounds bounded; per-card LFSR wake-key verified.
- **sysfs `resources` write CAP_SYS_ADMIN** — `disable`/`activate`/`set IRQ`/`set IO`/`set MEM`/`set DMA` commands require CAP_SYS_ADMIN.

Rationale: PNP resource assignment decides which IRQ lines, DMA channels, and IO/MMIO ranges a legacy device occupies — a misarbitered range silently maps a device's MMIO over kernel `.rodata` or over an already-claimed PCI BAR. The PNPBIOS path is structurally worse: it executes 16-bit firmware code with kernel privilege, bypassing W^X / kCFI / PaX guarantees, which is why locked-down kernels must refuse it. Bounded `_CRS` parsers, range-overlap rejection against `iomem_resource` and the kernel image, PNPBIOS LOCKDOWN gating + CAP_SYS_RAWIO, refcount-overflow trapping on `pnp_dev`, and kCFI on `pnp_protocol` dispatch turn PnP from "trust whatever the firmware says" into a validated, bounded enumeration where every assignment is policed before activation.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- ACPI namespace walker / ACPICA core (`drivers/acpi/`) — separate Tier-3.
- ISAPNP serial-isolation hardware details — Tier-4 `isapnp/`.
- PNPBIOS 16-bit thunk implementation specifics (`bioscalls.c` asm) — Tier-4.
- Per-PnP-driver clients (`serial_pnp`, `parport`, `floppy`, legacy sound) — Tier-4 each.
- Implementation code.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `arbiter_no_overlap` | EXCLUSIVITY | every `pnp_assign_resources` chosen alternative is disjoint from kernel image, `iomem_resource.busy`, `ioport_resource.busy`, and previously-assigned PnP devices. |
| `crs_parse_bounded` | OOB | `_CRS`/`_PRS` parser bounded by descriptor-length; refuse malformed. |
| `pnpbios_thunk_lockdown` | LOCKDOWN | `bioscalls.c` entry refused when `kernel_locked_down(LOCKDOWN_PNPBIOS)` is true. |
| `pnp_dev_no_uaf` | UAF | `pnp_dev` refcount drained before `pnp_protocol.disable` runs; remove ordered after driver detach. |

### Layer 2: TLA+

`models/pnp/arbiter.tla` (parent-declared): proves that multi-device assignment over a shared resource pool (e.g. two ISAPNP devices contending for the same IRQ) is conflict-free; the chosen alternative for each device is mutually disjoint from every other device's chosen alternative.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `pnp_assign_resources(dev, depnum)` post: each entry in `dev.res.resource[]` matches some entry in `dev.res.option[depnum]` AND passes `pnp_range_reserved` | `PnpDev::assign_resources` |
| `pnp_activate_dev(dev)` post: `dev.active == true` AND firmware state reflects `dev.res.resource[]` | `PnpDev::activate` |
| `pnpacpi_parse_resource_option_data(dev)` post: every parsed `_PRS` entry maps to a `pnp_option` with valid `(start, end, align, size)` constraints | `pnpacpi::Parser::parse_resource_option_data` |
| ISAPNP enumeration post: every emitted `pnp_dev` has a unique CSN | `isapnp::Protocol::enumerate` |

### Layer 4: Verus/Creusot functional

PnP discovery → resource arbiter → activation → driver probe; on disable, firmware state cleared → arbiter releases the range → next device may claim it. Encoded as a Verus refinement of an abstract two-phase assign/activate protocol with no observable inconsistency between `dev.res.resource[]` and firmware state.
