# Tier-3: drivers/parport/{share,ieee1284,procfs}.c — Parallel-port framework (parport bus + IEEE 1284 negotiation + procfs)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/parport/share.c
  - drivers/parport/ieee1284.c
  - drivers/parport/ieee1284_ops.c
  - drivers/parport/procfs.c
  - drivers/parport/daisy.c
  - drivers/parport/probe.c
  - include/linux/parport.h
  - include/uapi/linux/parport.h
-->

## Summary

The `parport` core is the generic parallel-port framework: hardware drivers (`parport_pc`, `parport_serial`, `parport_amiga`, `parport_atari`, `parport_gsc`, `parport_ip32`, `parport_sunbpp`, `parport_cs`) register a `struct parport` describing a physical port; client drivers (`lp`, `ppdev`, `plip`, scanner/zip/cd-rom legacy drivers) register a `struct pardevice` and claim exclusive access via the share manager. `share.c` is the framework core (port registration, device claim/release, IRQ glue, daisy-chain enumeration through `daisy.c`+`probe.c`), `ieee1284.c` is the IEEE 1284 phase-negotiation state machine (Compatibility, Nibble, Byte, ECP, EPP modes), and `procfs.c` exposes the long-standing `/proc/sys/dev/parport/parport%u/` controls.

Sources: `share.c` (~1214 lines, port + device lifecycle + share manager + IRQ dispatch), `ieee1284.c` (~789 lines, negotiate/terminate state machine, transfer dispatch), `ieee1284_ops.c` (per-mode software-bitbang fallbacks), `procfs.c` (~597 lines), `daisy.c` (1284.3 daisy-chain), `probe.c` (1284 device-ID parse).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct parport` (`include/linux/parport.h`) | per-physical-port control block | `drivers::parport::Port` |
| `struct pardevice` (`include/linux/parport.h`) | per-client claim handle | `drivers::parport::ParDevice` |
| `struct parport_operations` | hardware-driver vtable (write_data / read_data / write_control / frob_control / read_status / enable_irq / disable_irq / data_forward / data_reverse / mode-specific transfer fns) | `drivers::parport::PortOps` |
| `parport_register_port(base, irq, dma, ops)` / `parport_remove_port(port)` | hardware-driver port registration | `Port::register` / `_remove` |
| `parport_announce_port(port)` / `parport_del_port(port)` | publish/unpublish to clients (triggers `parport_driver.match_port`) | `Port::announce` / `_delete` |
| `parport_register_dev_model(port, name, par_dev_cb, id)` | new-style device registration (since 2013) | `ParDevice::register` |
| `parport_unregister_device(pardev)` | legacy device unregistration | `ParDevice::unregister` |
| `__parport_register_driver(driver, owner, mod_name)` / `parport_unregister_driver(driver)` | parport_bus driver registration | `ParportDriver::register` / `_unregister` |
| `parport_claim(pardev)` / `parport_claim_or_block(pardev)` / `parport_release(pardev)` | per-device exclusive-access acquire/release | `ParDevice::claim` / `_claim_or_block` / `_release` |
| `parport_find_base(base)` / `parport_find_number(num)` | port lookup by I/O base or `parport%u` index | `Port::find_base` / `_find_number` |
| `parport_get_port(port)` / `parport_put_port(port)` | refcount | `Port::get` / `_put` |
| `parport_negotiate(port, mode)` | IEEE 1284 enter-mode | `Port::negotiate` |
| `parport_write(port, buf, len)` / `parport_read(port, buf, len)` | mode-dispatched bulk transfer | `Port::write` / `_read` |
| `parport_set_timeout(pardev, inactivity)` | per-device transfer timeout | `ParDevice::set_timeout` |
| `parport_irq_handler(irq, dev_id)` | shared top-half | `Subsystem::irq_handler` |
| `parport_ieee1284_interrupt(port)` | IEEE1284 phase IRQ callback into mode-specific code | `Port::ieee1284_interrupt` |
| `parport_wait_event(port, timeout)` | wait for IRQ wakeup with timeout | `Port::wait_event` |
| `parport_register_dev_model` callbacks: `preempt` / `wakeup` / `irq_func` | optional kick-out preempt + claim-on-wakeup | `ParDeviceCallbacks` |
| `parport_daisy_init(port)` / `parport_daisy_select(port, daisy, mode)` | IEEE 1284.3 daisy-chain enumerate + select | `Port::daisy_init` / `_daisy_select` |

## Compatibility contract

REQ-1: Per-port abstraction matches `parport_operations`: hardware drivers implement at minimum `write_data`, `read_data`, `read_status`, `write_control`, `frob_control`, `enable_irq`, `disable_irq`, `data_forward`, `data_reverse`, plus per-mode `compat_write_data`, `nibble_read_data`, `byte_read_data`, `epp_{read,write}_{data,addr}`, `ecp_{read,write}_{data,addr}`.

REQ-2: Per-port share manager: one `pardevice` holds the port at a time (`port.cad` — current-active-device); claim is via `parport_claim` (non-blocking) or `parport_claim_or_block` (sleeps); release wakes the next waiter from `port.waithead` FIFO.

REQ-3: Per-device cooperative preemption: if a `pardevice` registered a `preempt` callback, another client may force it off the port mid-transfer; the preempted client is expected to checkpoint state and return to the queue.

REQ-4: IEEE 1284 negotiation: `parport_negotiate(port, IEEE1284_MODE_*)` walks the spec sequence (assert nSelectIn+nAutoFd, wait nAck, write extensibility byte, wait peripheral confirm); failure reverts to compatibility mode and returns -EOPNOTSUPP.

REQ-5: IEEE 1284 modes supported: COMPAT (output-only Centronics), NIBBLE (reverse 4-bit), BYTE (reverse 8-bit), ECP (DMA-capable forward+reverse with FIFO), EPP (block I/O with hardware handshake); per-port `port.modes` bitmask reports HW support.

REQ-6: Software-only fallback in `ieee1284_ops.c`: every mode has a bitbang implementation usable when the host bridge lacks ECP/EPP HW (e.g. PCI parport cards without 1284 FIFO).

REQ-7: Per-device timeout: `parport_set_timeout(pardev, jiffies)` sets the inactivity bound for `wait_event` / `wait_peripheral`; -ETIMEDOUT returned to client if not satisfied.

REQ-8: `/proc/sys/dev/parport/parport%u/` exposes `base-addr`, `irq`, `dma`, `modes`, `devices`, `autoprobe*`, `spintime` controls with appropriate r/w gating per file mode.

REQ-9: Daisy-chain (IEEE 1284.3): up to 4 devices on one physical port; per-daisy address selected by 1284.3 enter-select sequence; `port->daisy` field tracks active daisy address.

REQ-10: 1284 device-ID parse: `parport_device_id(port, daisy, buffer, len)` returns the IEEE 1284 1284-ID string (MFG, MDL, CMD, DES, CLS), used by `lp` for printer-aware behavior and by udev for hotplug match.

REQ-11: Shared IRQ: per-port IRQ is hooked via `request_irq(irq, parport_irq_handler, IRQF_SHARED, port->name, port)`; the top-half dispatches to `port.cad`'s `irq_func` callback.

REQ-12: Suspend/resume: per hardware driver; the core does not assume any state — clients re-claim on resume.

## Acceptance Criteria

- [ ] AC-1: Modprobe `parport_pc` on a system with a legacy parallel port; `/sys/bus/parport/devices/parport0/` appears with correct `base-addr` and `modes`.
- [ ] AC-2: Modprobe `lp` — `/dev/lp0` is created, `echo hello > /dev/lp0` to a connected printer prints (or returns -EIO if offline).
- [ ] AC-3: Modprobe `ppdev` — `/dev/parport0` chardev opens; userspace `ppdev` ioctls (`PPCLAIM`, `PPRELEASE`, `PPSETMODE`, `PPWDATA`, `PPRDATA`, `PPNEGOT`) work.
- [ ] AC-4: Two clients on one port (`lp` + `ppdev`) properly serialize via share manager; preempt callbacks fire if registered.
- [ ] AC-5: `parport_negotiate(port, IEEE1284_MODE_ECP)` returns 0 on ECP-capable HW; falls back to compatibility on non-1284 peripherals.
- [ ] AC-6: 1284 device-ID readable via `cat /proc/sys/dev/parport/parport0/autoprobe` for IEEE 1284-aware peripheral.
- [ ] AC-7: PLIP networking between two hosts via crossover cable: ifconfig `plip0 up`, ping over IEEE 1284 nibble mode.
- [ ] AC-8: PCMCIA `parport_cs` card insert produces `parport1` then `lp1`.

## Architecture

`Port` is the per-physical-port runtime:

```
struct Port {
  base: ResourceSize,            // ISA I/O base (e.g. 0x378)
  base_hi: ResourceSize,         // ECR window (e.g. 0x778)
  size: ResourceSize,
  name: KString,
  irq: u32,                      // PARPORT_IRQ_NONE if PIO-only
  dma: u32,                      // PARPORT_DMA_NONE if PIO-only
  modes: ParportModes,           // bitmask PCSPP | TRISTATE | EPP | ECP | DMA | SAFEININT
  number: u32,                   // parport%u
  ops: &'static PortOps,
  ieee1284: Ieee1284State,       // mode, phase, peripheral ID
  cad: AtomicPtr<ParDevice>,     // current active device
  daisy: i8,                     // 1284.3 daisy address (-1 if none)
  muxport: i8,                   // for parport_pc 1284-mux
  waithead: Mutex<Vec<Arc<ParDevice>>>,
  pardevice_list: Mutex<Vec<Arc<ParDevice>>>,
  probe_info: [Probe; 5],        // per-daisy
  spintime: AtomicU32,            // busy-wait spin in usec
  refcount: Refcount,
  dev: Device,                   // parport_bus parent
  bus_list: SpinLock<Vec<Weak<Port>>>,
}

struct ParDevice {
  name: KString,
  port: Arc<Port>,
  preempt: Option<fn(*mut ())>,
  wakeup: Option<fn(*mut ())>,
  irq_func: Option<fn(*mut ()) -> IrqReturn>,
  flags: u32,                    // PARPORT_DEV_TRAN | _LURK | _EXCL
  ctx: NonNull<()>,
  timeout: AtomicI64,             // jiffies
  state: AtomicU32,
  waiting: AtomicU32,             // 1 if in waithead
  next: Option<Arc<ParDevice>>,
  prev: Option<Weak<ParDevice>>,
}
```

Hardware-driver port registration `Port::register`:
1. `parport_register_port(base, irq, dma, ops)` allocs `Port`, assigns `number` from idr, sets ops.
2. Driver fills in `modes`, `base_hi`, calls `parport_announce_port(port)`.
3. `announce` runs through `parport_drivers` (new-style `parport_driver.match_port`) which gives each client driver a chance to bind.
4. `device_add(&port.dev)` under `parport_bus_type` (parent `dev`).

Per-device claim `ParDevice::claim`:
1. CAS `port.cad` from NULL → `pardev`; on success, return 0.
2. On contention: if current `cad.preempt` exists, call it; if it returns "I'm done", retry CAS.
3. On still-contended: push `pardev` onto `port.waithead`, return -EAGAIN (non-blocking) or sleep (`_claim_or_block`).

Per-device release `ParDevice::release`:
1. Assert `port.cad == pardev`.
2. Pop next waiter from `waithead`; CAS `port.cad` to next; invoke its `wakeup` callback (which typically re-claims).
3. If no waiter: `port.cad = NULL`.

IEEE 1284 negotiation `Port::negotiate(mode)`:
1. Assert hosts is in `IEEE1284_PH_FWD_IDLE`; if not, `parport_ieee1284_terminate(port)` first.
2. Write extensibility request byte to data lines.
3. Pulse nSelectIn high, nAutoFd low.
4. Wait `T_event` (1 ms) for peripheral nAck assert.
5. Read PE/Select/nFault status to confirm peripheral acceptance.
6. On success: `port.ieee1284.mode = mode`; `port.ieee1284.phase = IEEE1284_PH_HBUSY_DAVAIL` (or per-mode).
7. On failure: `parport_ieee1284_terminate` to revert; return -EOPNOTSUPP.

Mode-dispatched transfer `Port::write(buf, len)`:
1. Switch on `port.ieee1284.mode`: `parport_ieee1284_write_compat` / `_ecp_write_data` / `_epp_write_data`.
2. Each per-mode op uses `port.ops` if HW-accelerated (FIFO push, DMA setup) or `ieee1284_ops.c` bitbang fallback.
3. Block on `parport_wait_peripheral(port, mask, val)` between byte/block boundaries with per-device timeout.

ECP DMA path:
1. `port.ops.ecp_write_data(port, buf, len, 0)` programs ISA DMA channel (typically DMA 3) to push from kernel buffer to ECP FIFO.
2. ECP FIFO empties to peripheral while CPU is free.
3. DMA completion IRQ → `parport_irq_handler` → `port.cad.irq_func` → wake waiter.

Daisy-chain enumerate `Port::daisy_init`:
1. Issue IEEE 1284.3 daisy-init sequence (negotiate to BYTE mode with daisy ID 0x55).
2. For daisy in 0..4: select via 1284.3 select command, read 1284 device-ID; on success, record into `port.probe_info[daisy]`.
3. `parport_daisy_select(port, daisy, mode)` later switches active daisy address before per-device transfer.

procfs interface (`/proc/sys/dev/parport/parport%u/`):
- Per-file `ctl_table` registered via `register_sysctl_table`.
- `base-addr`, `irq`, `dma`: read-only.
- `modes`: read-only (space-separated PCSPP/TRISTATE/EPP/ECP/DMA).
- `devices`: lists registered `pardevice` names with `+` marking `cad`.
- `autoprobe[0..3]`: per-daisy 1284 device-ID strings.
- `spintime`: writable busy-wait spin (μs) for nAck poll.

## Hardening

- `parport_claim` and `_release` enforce single-writer on `port.cad`: client driver UB on transferring without claim.
- Share-manager `waithead` is bounded by per-`ParDevice` `waiting` flag — a device cannot be on the waitlist twice.
- IEEE 1284 negotiate has hard `T_event` (1 ms) timeouts at each phase transition; defense against stuck peripheral wedging the port.
- ECP DMA buffer length capped by `port.ops` per-driver implementation; the core does not let `len` exceed device-driver-declared max.
- 1284 device-ID parser bounds string length by `1024` and treats malformed (missing `;` separator) as empty.
- `parport_irq_handler` is `IRQF_SHARED`-safe: returns `IRQ_NONE` if `port.cad == NULL` or no `irq_func`.
- `ppdev` userspace ioctl path (separate Tier-3) gates `PPSETMODE`/`PPNEGOT`/`PPWDATA`/`PPRDATA` behind `CAP_SYS_RAWIO`.
- `parport_set_timeout` clamps to `[1, MAX_JIFFY_OFFSET]` to avoid wraparound math.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `parport`, `pardevice`, and `parport_device_info` 1284-ID buffers; ppdev `PPWDATA`/`PPRDATA` userspace buffers bounded.
- **PAX_KERNEXEC** — parport core (`share.c`, `ieee1284.c`, `ieee1284_ops.c`, `procfs.c`) in W^X kernel text; `parport_operations`, `parport_driver.{probe,detach,match_port}` vtables placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel stack on `parport_irq_handler` entry, `parport_claim_or_block` sleep path, IEEE 1284 negotiation, and ppdev ioctl entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `parport` (`refcount`), `pardevice` lifetime, and 1284-ID buffer refs; overflow trap defeats claim/release race UAF.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `parport`, `pardevice`, 1284 device-ID buffers, and per-port `probe_info` so prior-peripheral identity cannot bleed between port re-bindings.
- **PAX_UDEREF** — SMAP/PAN enforced on every ppdev ioctl + `/proc/sys/dev/parport` write entry; userspace pointer deref restricted to canonical helpers.
- **PAX_RAP / kCFI** — `parport_operations`, `parport_driver` callbacks, `pardevice.preempt`/`.wakeup`/`.irq_func`, and IEEE 1284-mode dispatch tables marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms disclosure of port I/O bases and `parport` pointers behind CAP_SYSLOG; `%p` neutered in parport tracepoints and `/proc/sys/dev/parport/parport%u/devices`.
- **GRKERNSEC_DMESG** — restrict 1284-negotiate-fail, IRQ-probe-fail, and share-manager-stall banners to CAP_SYSLOG.
- **CAP_SYS_RAWIO on /dev/parport*** — `ppdev` chardev ops (`PPSETMODE`, `PPNEGOT`, `PPWDATA`, `PPRDATA`, `PPCLAIM`, `PPDATADIR`) require CAP_SYS_RAWIO even when chardev is file-mode-accessible.
- **IEEE 1284 timing safety** — all wait-for-peripheral loops bounded by `T_event` + per-device timeout; refuse infinite busy-wait, defense against stuck peripheral causing CPU lockup.
- **IRQ-probe CAP_SYS_RAWIO** — `/proc/sys/dev/parport/parport%u/irq` and `autoprobe` writes that trigger live IRQ probing require CAP_SYS_RAWIO.
- **Share/release refcount discipline** — `parport_get_port`/`_put_port` and `pardevice` ref accounting validated such that a release cannot drop the last ref while `cad == self`.
- **Daisy-chain bounds** — 1284.3 daisy enumeration bounded to 4 children; refuse address > 3.
- **1284 device-ID sanitization** — `MFG:`/`MDL:`/`CMD:`/`DES:`/`CLS:` strings stripped of non-printable + shell metacharacters before being exposed via procfs or uevent.

Rationale: parallel ports route attacker-controlled IRQ lines + DMA buffers across a non-isolated bus. A miswritten share-manager release or a busy-wait without timeout silently turns a stuck printer into a CPU lockup. Bounded 1284 timing, kCFI-typed `parport_operations` dispatch, refcount-overflow trap on `pardevice` lifetime, and CAP_SYS_RAWIO on `/dev/parport*` plus IRQ probing convert the parport stack from "1990s trust" into a bounded enrolment surface even when legacy hardware is still attached.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-hardware-driver bridges (`parport_pc`, `parport_serial`, `parport_amiga`, `parport_atari`, `parport_gsc`, `parport_ip32`, `parport_sunbpp`, `parport_cs`) — Tier-4 each.
- `lp` line-printer client (`drivers/char/lp.c`) — separate Tier-3.
- `ppdev` userspace chardev (`drivers/char/ppdev.c`) — separate Tier-3.
- `plip` networking client (`drivers/net/plip/`) — separate Tier-3.
- Implementation code.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cad_single_writer` | EXCLUSIVITY | `port.cad` has at most one non-NULL value at any time; only `claim`/`release` modify it. |
| `negotiate_bounded` | TERMINATION | `parport_negotiate` returns within `T_event` * phase-count; no infinite loop on stuck peripheral. |
| `pardevice_no_uaf` | UAF | `pardevice` reference held by `port.waithead` and `port.pardevice_list` outlives any access via `port.cad`. |
| `daisy_bounded` | OOB | daisy address index in `[0,3]`; refuse access to `port.probe_info[4..]`. |

### Layer 2: TLA+

`models/parport/share_manager.tla` (parent-declared): proves FIFO fairness of `waithead`, no two devices simultaneously holding `cad`, no lost release wakeup, no preempt-during-claim race.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ParDevice::claim` post: returns OK only if CAS NULL->self succeeded on `port.cad` | `ParDevice::claim` |
| `ParDevice::release` post: `port.cad` is either the next waiter or NULL | `ParDevice::release` |
| `Port::negotiate(mode)` post: `port.ieee1284.mode == mode` on OK return, or `mode == COMPAT` on failure | `Port::negotiate` |
| 1284-ID parse invariant: returned string is NUL-terminated and length <= 1024 | `parport_device_id` |

### Layer 4: Verus/Creusot functional

claim → exclusive access for the duration → release wakes next waiter → next waiter gains exclusive access; encoded as Verus refinement of an abstract mutex queue ensuring no lost wakeup and no double-grant.
