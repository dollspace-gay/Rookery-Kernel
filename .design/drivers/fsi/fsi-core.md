# Tier-3: drivers/fsi/fsi-core.c + fsi-master-* — FSI (FRU Support Interface) bus subsystem

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/fsi/fsi-core.c
  - drivers/fsi/fsi-master.h
  - drivers/fsi/fsi-slave.h
  - drivers/fsi/fsi-master-aspeed.c
  - drivers/fsi/fsi-master-gpio.c
  - drivers/fsi/fsi-master-hub.c
  - drivers/fsi/fsi-master-ast-cf.c
  - drivers/fsi/fsi-master-i2cr.c
  - drivers/fsi/fsi-occ.c
  - drivers/fsi/fsi-sbefifo.c
  - drivers/fsi/fsi-scom.c
  - include/linux/fsi.h
  - include/uapi/linux/fsi.h
-->

## Summary

FSI (FRU Support Interface) is IBM's two-wire bus subsystem used as the out-of-band debug + service interface for POWER CPUs and OpenCAPI accelerators. The BMC (typically an Aspeed AST2500/AST2600 in an OpenPOWER server) bit-bangs an FSI master that talks to one or more Slaves on the host CPUs; each Slave fans out into addressable Engines (SCOM, SBEFIFO, OCC, I2C-master, GPIO, etc.) which expose register-level access to the host SoC. FSI is the path by which OpenBMC reads host CPU sensors, programs SCOM registers during host bring-up, drives the Self-Boot-Engine (SBE) firmware FIFO, and exfiltrates RAS events.

The architecture is master-slave-engine layered. A `fsi_master` is registered by a low-level driver (`fsi-master-aspeed.c` for AT&T-style Aspeed FSI master hardware, `fsi-master-gpio.c` for bit-banged GPIO fallback, `fsi-master-hub.c` for cascaded FSI hub engines, `fsi-master-ast-cf.c` for ColdFire-coprocessor offload, `fsi-master-i2cr.c` for I2C-responder master). Each registered master scans its links via `fsi_master_scan(master)`; for each link it issues a TERM/BREAK + reads the slave ID register, instantiates a `fsi_slave`, then walks the slave's engine table (CFAM registers) to instantiate `fsi_device` engines. Engine drivers (`fsi-scom.c`, `fsi-occ.c`, `fsi-sbefifo.c`) match on `engine_type + version` and expose char-device or sysfs nodes (e.g. `/dev/scom0`, `/dev/occ1`, `/dev/sbefifo0`) for OpenBMC userland.

This Tier-3 covers `fsi-core.c` (~1500 lines: bus registration, master scan, slave create, engine walk, per-engine address validation) and the `fsi_master`/`fsi_slave` API surface used by all in-tree masters.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `fsi_bus_type` | device-model bus | `drivers::fsi::Bus` |
| `struct fsi_master` | per-master control block | `drivers::fsi::Master` |
| `struct fsi_slave` | per-slave record | `drivers::fsi::Slave` |
| `struct fsi_device` | per-engine record (FSI-engine "device") | `drivers::fsi::EngineDev` |
| `struct fsi_driver` | per-engine-driver registration | `drivers::fsi::EngineDriver` |
| `fsi_master_register(master)` / `_unregister(master)` | master registration | `Master::register` / `_unregister` |
| `fsi_master_scan(master)` (and `fsi_slave_init` per-link) | per-link probe + slave create | `Master::scan` |
| `fsi_master_read(master, link, id, addr, val, sz)` / `_write(...)` | master-level raw read/write | `Master::read` / `_write` |
| `fsi_master_break(master, link)` | issue BREAK to reset slave | `Master::break_` |
| `fsi_master_term(master, link, id)` | issue TERM to abort op | `Master::term` |
| `fsi_slave_read(slave, addr, val, sz)` / `_write(...)` | per-slave address-space read/write | `Slave::read` / `_write` |
| `fsi_slave_claim_range(slave, addr, sz)` / `_release_range(...)` | engine-allocator: claim region of CFAM | `Slave::claim_range` / `_release_range` |
| `fsi_device_read(dev, addr, val, sz)` / `_write(...)` / `_peek(...)` | per-engine read/write | `EngineDev::read` / `_write` / `_peek` |
| `fsi_driver_register(drv)` / `_unregister(drv)` | engine-driver registration | `EngineDriver::register` / `_unregister` |
| `fsi_slave_handle_error(slave, write, addr, sz)` | error-recovery: SISC/SSTAT read + clear + BREAK + restore | `Slave::handle_error` |
| `fsi_slave_report_and_clear_errors(slave)` | per-slave error status drain | `Slave::report_and_clear_errors` |
| `fsi_slave_set_smode(slave)` | program SMODE register on slave | `Slave::set_smode` |

## Compatibility contract

REQ-1: ABI `include/linux/fsi.h` for in-kernel + `include/uapi/linux/fsi.h` for char-device engines must remain stable across kernel versions; per-engine UAPI (SCOM ioctls, OCC ioctl, SBEFIFO ioctl) preserved.

REQ-2: FSI protocol bit-timing per-link is master-driver-specific (Aspeed FSI master vs GPIO bit-bang vs ColdFire coprocessor); core does not arbitrate timing — only logical reads/writes via `master.ops.read`/`_write`.

REQ-3: Per-link slave id space — at most 4 slaves per link per spec; per-slave address space 22-bit CFAM (4 MiB). Slave registers live at `FSI_SLAVE_BASE = 0x800` upward.

REQ-4: Engine enumeration via per-slave Engine table at `FSI_PEEK_BASE = 0x410`; each engine descriptor (4 bytes) carries `engine_type`, `version`, `crc`, `addr_range`. Core walks this table during `fsi_slave_init` and instantiates `fsi_device` for each non-zero entry.

REQ-5: Per-engine `id_table` matches `(engine_type, version)`; an engine driver may bind via `engine_type` wildcard (`version == 0` matches any). Probe = `fsi_drv.probe(fsi_dev)`; remove = `fsi_drv.remove(fsi_dev)`.

REQ-6: Error recovery — every transfer error returns one of `FSI_SLAVE_ERR_{...}` codes; `fsi_slave_handle_error` issues BREAK, restores SMODE, drains errors, retries up to `slave_retries = 2`. If retries fail, returns to caller with original error.

REQ-7: Char-device engines (SCOM/OCC/SBEFIFO) allocate minor numbers from per-driver IDA seeded against `fsi_minor_ida` + `fsi_base_dev`; node naming `/dev/scomN`, `/dev/occN`, `/dev/sbefifoN`.

REQ-8: Master probe path: low-level driver (e.g. `fsi-master-aspeed`) sets `master.dev`, `master.links`, `master.ops`, then calls `fsi_master_register`. Master is hot-pluggable; unregister tears down all slaves + engines.

REQ-9: `discard_errors` module param disables error reporting on best-effort masters (GPIO bit-bang where occasional CRC mismatches are expected during host reset windows).

REQ-10: Per-slave `claim_range` / `release_range` is the resource-allocator for engine drivers — overlapping claims fail with `-EBUSY`.

## Acceptance Criteria

- [ ] AC-1: OpenBMC AST2600 boot with `fsi-master-aspeed` enabled — `/sys/bus/fsi/devices/` populated with slaves + engines on host POWER10 reset cycle.
- [ ] AC-2: `pdbg` userspace tool reads SCOM register via `/dev/scom0` round-trips correctly.
- [ ] AC-3: SBE firmware load via `/dev/sbefifo0` completes and host POWER CPU IPLs successfully.
- [ ] AC-4: OCC sensor poll via `/dev/occ0` returns plausible CPU temperatures.
- [ ] AC-5: Forced BREAK during transfer — `fsi_slave_handle_error` recovers; second op succeeds.
- [ ] AC-6: Hot-unregister master while engine driver bound — engine driver `remove` runs, devices removed.
- [ ] AC-7: `fsi-master-gpio` bit-banged fallback: every error returns a defined errno; no UAF, no infinite loop.

## Architecture

`Master` lives in `drivers::fsi::Master`:

```
struct Master {
  dev: &Device,
  idx: u32,                       // master_ida allocated
  n_links: u8,
  flags: u32,                     // FSI_MASTER_FLAG_* (e.g. SWCLOCK)
  ops: &'static MasterOps,
  scan_lock: Mutex<()>,
  slaves: Mutex<Vec<Arc<Slave>>>, // per-link
}

struct MasterOps {
  read:   fn(&Master, link: u8, id: u8, addr: u32, val: &mut [u8]) -> Result<()>,
  write:  fn(&Master, link: u8, id: u8, addr: u32, val: &[u8]) -> Result<()>,
  term:   Option<fn(&Master, link: u8, id: u8) -> Result<()>>,
  send_break: fn(&Master, link: u8) -> Result<()>,
  link_enable:  Option<fn(&Master, link: u8, enable: bool) -> Result<()>>,
  link_config:  Option<fn(&Master, link: u8, t_send: u8, t_echo: u8) -> Result<()>>,
}

struct Slave {
  master: NonNull<Master>,
  dev: KBox<Device>,
  cfam_id: u32,
  link: u8,
  id: u8,
  size: u32,                       // 4 MiB CFAM size
  t_send_delay: u8,
  t_echo_delay: u8,
  ranges: Mutex<Vec<Range<u32>>>,  // claim_range allocator
  engine_idx: AtomicU32,
}

struct EngineDev {
  dev: KBox<Device>,                // bus = fsi_bus_type
  slave: NonNull<Slave>,
  engine_type: u8,
  version: u8,
  unit: u8,
  addr: u32,                       // CFAM offset of engine register block
  size: u32,
  npes: u32,
}
```

Master register `fsi_master_register(master)`:
1. `ida_alloc(&master_ida)` → assign master idx.
2. Set `master.dev.bus = &fsi_bus_type`.
3. `device_register(&master.dev)`.
4. Per-link: call `fsi_master_scan(master)`.

Per-master scan `fsi_master_scan(master)`:
1. For each link in `master.n_links`:
   - `fsi_master_break(master, link)` — reset slave.
   - `fsi_master_read(master, link, 0, FSI_SLAVE_SMODE, ...)` — read slave id register.
   - Validate CFAM id present; if not, skip link.
   - For each slave id on link (1..4): try `fsi_slave_init(master, link, slave_id)`.

Per-slave init `fsi_slave_init`:
1. Allocate `fsi_slave`, populate `cfam_id`, `link`, `id`, `t_send_delay = 16`, `t_echo_delay = 16`.
2. `fsi_slave_set_smode(slave)` — write SMODE with delays + sid.
3. `device_register(&slave.dev)` with `bus = &fsi_bus_type`.
4. `fsi_slave_scan(slave)` — walk engine table at `FSI_PEEK_BASE`, allocate `fsi_device` for each engine entry; per-entry `engine_type`/`version` from descriptor; `claim_range(addr, size)`.

Per-transfer error path `fsi_slave_handle_error(slave, write, addr, size)`:
1. Read SISC (Slave Interrupt Status) + SSTAT (Slave Status) via raw `fsi_master_read`.
2. Log status if not in `discard_errors` mode.
3. Clear SISC bits.
4. Issue `fsi_master_break(master, link)`.
5. Restore SMODE via `fsi_slave_set_smode`.
6. `fsi_slave_report_and_clear_errors` again to drain.
7. Return original error to caller; caller retries up to `slave_retries = 2`.

Engine driver lifecycle: registered via `fsi_driver_register`; `fsi_bus_match` walks `id_table` matching `(engine_type, version)`; `fsi_probe(dev)` calls into `fsi_drv->probe(fsi_dev)`; per-engine alloc per-instance minor + `cdev_add` (for char-device engines).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cfam_addr_oob` | OOB | every `fsi_slave_read/_write` validates `addr + sz <= slave.size` (4 MiB). |
| `link_id_oob` | OOB | every master op validates `link < master.n_links` and `id < 4`. |
| `claim_range_no_overlap` | OVERLAP | per-slave `ranges` invariant: no two claims overlap. |
| `slave_refcount_no_uaf` | UAF | `Arc<Slave>` outlives all engines; engine remove decrements before slave free. |

### Layer 2: TLA+

`models/fsi/error_recovery.tla` — proves the BREAK/restore/retry loop bounded: after `slave_retries` attempts it terminates and returns failure; never blocks forever on a wedged slave.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `fsi_slave_calc_addr` post: returns correct `(id, addr)` for given slave + slave-local addr | `Slave::calc_addr` |
| `fsi_slave_handle_error` post on success: SISC cleared, SMODE restored, slave armed | `Slave::handle_error` |
| `claim_range` post: range added to ordered list iff no overlap; release removes exact range | `Slave::claim_range` |
| Master register / unregister symmetry: all slaves + engines torn down | `Master::register`/`_unregister` |

### Layer 4: Verus functional

Per-engine read/write through master.ops with intervening BREAK preserves data semantics: post-BREAK + restore, an `fsi_device_read` returns the same value as if the BREAK had not occurred (modulo the engine's own state). Encoded as Verus refinement over the master+slave transcript.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

fsi-core specific reinforcement:

- **Per-slave address-range validation** — `fsi_slave_calc_addr` rejects addresses past 4 MiB CFAM size.
- **`slave_retries = 2` ceiling** — bounded retry on error before giving up; defense against wedged-slave infinite loop.
- **Engine-table CRC validated** — engine descriptors must pass CRC before instantiation.
- **`discard_errors` opt-in only** — best-effort masters explicitly opt in; default is strict error reporting.
- **IDA-based minor allocation** — per-engine-driver IDA seeded from `fsi_minor_ida` prevents minor collision across char-device engines.
- **Master scan serialized by `scan_lock`** — defense against concurrent hot-add of two masters racing on the same link.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `fsi_master`, `fsi_slave`, `fsi_device`, and per-engine private state; SCOM/OCC/SBEFIFO ioctl payloads size-validated on the engine-driver side.
- **PAX_KERNEXEC** — FSI core text + `fsi_bus_type`, per-engine-driver `fsi_driver` op-vectors, and per-master `MasterOps` in W^X kernel text; `__ro_after_init` for the bus, the engine-driver `id_table` pointers, and the global `master_ida`/`fsi_minor_ida` controls.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `fsi_master_scan`, `fsi_slave_init`, `fsi_slave_handle_error`, `fsi_slave_read`/`_write`, `fsi_device_read`/`_write`, and per-master IRQ entries (Aspeed master IRQ thread).
- **PAX_REFCOUNT** — saturating `refcount_t` on `fsi_master`, `fsi_slave`, `fsi_device`, and per-engine cdev references; overflow trap defeats unregister-during-transfer race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-slave CFAM scratch buffers, engine descriptor records on `_release_range`, and per-engine private data on remove so stale SCOM payloads / SBE firmware fragments cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every SCOM / OCC / SBEFIFO ioctl entry into engine drivers (`/dev/scom*`, `/dev/occ*`, `/dev/sbefifo*`).
- **PAX_RAP / kCFI** — per-master `MasterOps` (`read`, `write`, `send_break`, `term`, `link_enable`, `link_config`) and per-engine-driver `probe`/`remove` callbacks marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-master MMIO pointer / Aspeed FSI base disclosure behind CAP_SYSLOG; suppress `%p` in slave error tracepoints.
- **GRKERNSEC_DMESG** — restrict slave error (SISC/SSTAT dump), BREAK-on-error, OCC/SBE timeout, and scan-fail banners to CAP_SYSLOG so attackers cannot fingerprint POWER host bring-up via dmesg.
- **Master CAP_SYS_RAWIO** — FSI gives the BMC full debug access to the host CPU (SCOM = raw host CPU register R/W); opening `/dev/scom*`, `/dev/occ*`, or `/dev/sbefifo*` requires CAP_SYS_RAWIO and is gated by lockdown in `LOCKDOWN_HIBERNATION` confidentiality tier.
- **Slave-engine allowlist** — engine descriptor `engine_type` matched against per-platform allowlist (SCOM, SBEFIFO, OCC, FSI-I2C only by default); unknown engine types not instantiated.
- **Scan-bus CAP_SYS_ADMIN** — sysfs writes that force a rescan of a master (`/sys/bus/fsi/devices/.../rescan`) require CAP_SYS_ADMIN.
- **Address-range bounded** — every `fsi_slave_read`/`_write` validates `addr + sz <= slave.size`; every engine-driver read/write validates against per-engine `addr/size` claim.
- **Error-handling rate-limited** — repeated SISC drains rate-limited under CAP_SYSLOG to prevent dmesg flooding on wedged hardware.
- **Engine descriptor CRC** — per-engine table entry CRC validated before `fsi_create_device`; refuse instantiation on CRC fail.

Rationale: FSI is the OpenBMC's full-throttle host-CPU debug bus — SCOM access can rewrite the host PowerPC's machine state, SBEFIFO can replace boot firmware. A buggy engine descriptor, a missing CRC check, or an unauth'd `/dev/scom*` open is a complete bypass of host integrity. CAP_SYS_RAWIO on engine chardevs, signed engine allowlists, address-range bounds, RAP/kCFI on master/engine ops, refcount-overflow trapping, and rate-limited error reporting turn FSI from "BMC owns the host" into "BMC has audited, structurally-bounded host access."

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-master driver internals (`fsi-master-aspeed.c`, `fsi-master-gpio.c`, `fsi-master-hub.c`, `fsi-master-ast-cf.c`, `fsi-master-i2cr.c`) — each future Tier-3.
- Per-engine driver internals (`fsi-scom.c`, `fsi-occ.c`, `fsi-sbefifo.c`, `i2cr-scom.c`) — each future Tier-3.
- POWER chip SCOM register semantics — vendor IP, out of scope.
- 32-bit-only paths.
- Implementation code.
