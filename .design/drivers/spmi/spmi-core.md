# Tier-3: drivers/spmi/{spmi,spmi-devres}.c — SPMI bus framework (System Power Management Interface)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/spmi/spmi.c
  - drivers/spmi/spmi-devres.c
  - include/linux/spmi.h
-->

## Summary

The SPMI (System Power Management Interface, MIPI Alliance) framework — a two-wire serial bus standardized for low-power management between an Application Processor (AP) and one or more Power-Management Integrated Circuits (PMICs). Up to 4 masters and 16 slaves per bus (4-bit Unique Slave Identifier, USID); single-wire clock + bidirectional data; addresses 5-bit short / 8-bit / 16-bit "extended" register spaces; commands include RESET, SLEEP, WAKEUP, SHUTDOWN, and register Read/Write families (CMD_READ, CMD_WRITE, CMD_ZERO_WRITE, CMD_EXT_READ, CMD_EXT_WRITE, CMD_EXT_READL, CMD_EXT_WRITEL).

This Tier-3 covers `spmi.c` (~625 lines: bus_type, per-controller alloc/add/remove, per-device alloc/add/remove, OF-walk of slaves, R/W cmd helpers with length bounds, command helpers RESET/SLEEP/WAKEUP/SHUTDOWN) and `spmi-devres.c` (~64 lines: devm wrappers for controller and driver register). Master controllers live in `drivers/spmi/spmi-pmic-arb.c` (Qualcomm), `spmi-mtk-pmif.c` (MediaTek), `hisi-spmi-controller.c` (HiSilicon), `spmi-apple-controller.c` (Apple).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct spmi_controller` | per-bus master controller | `drivers::spmi::Controller` |
| `struct spmi_device` | per-slave device handle | `drivers::spmi::SpmiDevice` |
| `struct spmi_driver` | client driver registration | `drivers::spmi::SpmiDriver` |
| `spmi_controller_alloc(parent, size)` / `spmi_controller_add(ctrl)` / `_remove(ctrl)` | controller lifecycle | `Controller::alloc` / `_add` / `_remove` |
| `spmi_device_alloc(ctrl)` / `spmi_device_add(sdev)` / `_remove(sdev)` | per-slave lifecycle | `SpmiDevice::alloc` / `_add` / `_remove` |
| `__spmi_driver_register(sdrv, owner)` | client driver register | `SpmiDriver::register` |
| `spmi_register_read(sdev, addr, *buf)` | short-address (5-bit) 1-byte read | `SpmiDevice::register_read` |
| `spmi_ext_register_read(sdev, addr, *buf, len)` | 8-bit-addr extended read, ≤16 bytes | `SpmiDevice::ext_register_read` |
| `spmi_ext_register_readl(sdev, addr, *buf, len)` | 16-bit-addr extended long read, ≤8 bytes | `SpmiDevice::ext_register_readl` |
| `spmi_register_write(sdev, addr, data)` | short-address 1-byte write | `SpmiDevice::register_write` |
| `spmi_register_zero_write(sdev, data)` | CMD_ZERO_WRITE (7-bit reg-0) | `SpmiDevice::register_zero_write` |
| `spmi_ext_register_write(sdev, addr, *buf, len)` | 8-bit-addr extended write, ≤16 bytes | `SpmiDevice::ext_register_write` |
| `spmi_ext_register_writel(sdev, addr, *buf, len)` | 16-bit-addr extended long write, ≤8 bytes | `SpmiDevice::ext_register_writel` |
| `spmi_command_reset/_sleep/_wakeup/_shutdown(sdev)` | per-device state-transition cmds | `SpmiDevice::cmd_reset` / etc. |
| `spmi_find_device_by_of_node(np)` | look up `spmi_device` by DT node | `SpmiDevice::find_by_of_node` |
| `spmi_controller_set_drvdata` / `_get_drvdata` | per-controller private storage | `Controller::set_drvdata` / `_get_drvdata` |
| `ctrl->cmd / ->read_cmd / ->write_cmd` | per-controller master ops | `Controller::ops` |
| `spmi_cmd / spmi_read_cmd / spmi_write_cmd` | internal dispatch with tracepoints | `Controller::cmd_dispatch` |

## Compatibility contract

REQ-1: Bus topology — single APB-class controller per `spmi_controller`, up to `SPMI_MAX_SLAVE_ID` (=16) slaves addressed by 4-bit USID; device-name format `<ctrl_nr>-<USID:02x>`.

REQ-2: Per-command length bounds enforced by framework before controller dispatch:
- short read/write: 1 byte, 5-bit address (≤0x1F)
- ext read/write (8-bit addr): 1..16 bytes
- ext readl/writel (16-bit addr): 1..8 bytes

REQ-3: Controller ops vtable — `cmd(opcode, sid)`, `read_cmd(opcode, sid, addr, *buf, len)`, `write_cmd(opcode, sid, addr, *buf, len)`; framework rejects (-EINVAL) if vtable entry NULL or device type mismatch.

REQ-4: Per-controller dev type validated (`ctrl->dev.type == &spmi_ctrl_type`) on every helper to defeat per-bus typed-dispatch confusion; per-device type validated (`dev->type == &spmi_dev_type`) on remove walker.

REQ-5: OF enumeration — per child of controller DT node, `reg = <USID SPMI_USID>` required; reject USID ≥ SPMI_MAX_SLAVE_ID; reject reg[1] != SPMI_USID.

REQ-6: Per-device runtime-PM — `pm_runtime_get_noresume` + `_set_active` + `_enable` on probe; `_get_sync` + remove + disable on driver remove; per-device shutdown is best-effort (driver may omit).

REQ-7: Per-controller ida — `ctrl_ida` allocates bus number; name `spmi-<n>`; freed on controller release.

REQ-8: Per-driver match — both OF (`of_driver_match_device`) and per-driver-name (`strncmp(dev_name, drv->name, SPMI_NAME_SIZE=32)`).

REQ-9: Tracepoints — `trace_spmi_cmd`, `trace_spmi_read_begin/_end`, `trace_spmi_write_begin/_end` emitted per dispatch (CREATE_TRACE_POINTS owned by `spmi.c`).

REQ-10: Bus-register gating — `spmi_controller_add` refused with -EAGAIN until the postcore-initcall `spmi_init` completes; `is_registered` guard.

REQ-11: Per-driver shutdown — bus-level `spmi_drv_shutdown` invokes only when driver supplies `shutdown` hook; framework never sleeps in this path.

## Acceptance Criteria

- [ ] AC-1: Qualcomm SoC with `spmi-pmic-arb` master + PMIC slave probes successfully; `dmesg | grep spmi-` shows bus + slave registration.
- [ ] AC-2: PMIC register R/W via `regmap-spmi` works (read/modify/write a benign PMIC register such as a watchdog scratch).
- [ ] AC-3: SLEEP/WAKEUP cycle on PMIC succeeds without leaving pm_runtime imbalance.
- [ ] AC-4: USID-out-of-range DT entry rejected with error log; remaining slaves still probe.
- [ ] AC-5: Out-of-tree driver registers via `__spmi_driver_register`; bind succeeds via OF compatible or driver-name match.
- [ ] AC-6: Malformed ext-read with len=0 or len>16 returns -EINVAL.

## Architecture

```
struct Controller {
  dev: Device,                              // type = spmi_ctrl_type, bus = spmi_bus_type
  nr: u32,                                  // ida-allocated bus id, name "spmi-<nr>"
  cmd: Option<fn(&Self, u8 opcode, u8 sid) -> Result<()>>,
  read_cmd:  Option<fn(&Self, u8 opcode, u8 sid, u16 addr, *mut u8, len) -> Result<()>>,
  write_cmd: Option<fn(&Self, u8 opcode, u8 sid, u16 addr, *const u8, len) -> Result<()>>,
  drvdata: *mut (),                         // controller-private region trailing the struct
}

struct SpmiDevice {
  dev: Device,                              // type = spmi_dev_type
  ctrl: &Controller,
  usid: u8,                                 // 4-bit slave id, [0, SPMI_MAX_SLAVE_ID)
}
```

Controller-add `spmi_controller_add(ctrl)`:
1. Guard: `is_registered` must be true (bus already bus_register'd in `spmi_init`).
2. `device_add(&ctrl->dev)`.
3. If CONFIG_OF: walk DT children, parse `reg = <usid SPMI_USID>`, alloc `spmi_device` per child, set `usid`, `spmi_device_add`.

Per-slave add `spmi_device_add(sdev)`:
1. Set name `<ctrl->nr>-<usid:02x>`.
2. `device_add(&sdev->dev)` — invokes bus_type.match → bus_type.probe.

Per-command dispatch `spmi_read_cmd / spmi_write_cmd / spmi_cmd`:
1. Validate ctrl != NULL, ctrl->{cmd|read_cmd|write_cmd} != NULL, dev type matches.
2. Emit trace_spmi_*_begin.
3. Call vtable op.
4. Emit trace_spmi_*_end with `ret + buf + len`.

Helper bounds:
- `spmi_register_read/_write(addr)`: reject addr > 0x1F.
- `spmi_ext_register_read/_write(addr, buf, len)`: reject len==0 or len>16.
- `spmi_ext_register_readl/_writel(addr, buf, len)`: reject len==0 or len>8.

Driver register `__spmi_driver_register(sdrv, owner)`: set sdrv->driver.bus, set owner, driver_register.

Bus init: `spmi_init` (postcore_initcall) → `bus_register(&spmi_bus_type)` → `is_registered = true`. `spmi_exit` unregisters the bus.

devres (`spmi-devres.c`):
- `devm_spmi_controller_alloc(parent, size)` — alloc + auto-put on devm release.
- `devm_spmi_controller_add(parent, ctrl)` — add + auto-remove on devm release.
- `devm_spmi_driver_register(drv)` — register + auto-unregister via devm.

## Hardening

(Inherits row-1 features from `drivers/bus/00-overview.md` § Hardening.)

spmi-core specific reinforcement:

- **Per-command length bounds** rejected by framework helpers before controller op dispatch (5-bit addr ≤0x1F, ext ≤16, extl ≤8).
- **`dev.type` checks** on every cmd dispatch — defeats type confusion between controller and slave devices.
- **USID range bounded to [0, SPMI_MAX_SLAVE_ID)** at OF enumeration — rejects oversize USID with diag log.
- **`is_registered` guard** rejects controller-add before bus_register completes — defeats early-initcall race.
- **Per-controller ida freed in release** — `ctrl_ida` slot reusable only after refcount drops to zero.
- **Probe/remove pm_runtime balance** — `pm_runtime_get_noresume` + `_set_active` + `_enable` on probe; `_get_sync` + `_put_noidle` + `_disable` + `_set_suspended` + `_put_noidle` on remove; failure path on probe disables PM and decrements refcount.
- **`spmi_command_*` helpers serialize through `ctrl->cmd`** with tracepoints — every command observable for audit.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `spmi_controller` and `spmi_device` (sized via `kzalloc(sizeof + size)` for per-controller private region); read/write buffers passed by client drivers, never directly copied from userspace inside the framework.
- **PAX_KERNEXEC** — SPMI framework core in W^X kernel text; `bus_type`, `device_type` table, and per-controller `cmd`/`read_cmd`/`write_cmd` ops live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `spmi_read_cmd`, `spmi_write_cmd`, `spmi_cmd`, and `spmi_drv_probe`.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-`spmi_controller` and per-`spmi_device` `dev` refcount; pm_runtime get/put paired strictly across all probe/remove error branches; overflow trap defeats attach/detach race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `spmi_controller` and `spmi_device` slab objects so stale USID/private-region state cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every controller-driver entry; SPMI framework helpers themselves take only kernel-owned buffers.
- **PAX_RAP / kCFI** — master ops vtable (`cmd`/`read_cmd`/`write_cmd`), `spmi_driver` ops (`probe`/`remove`/`shutdown`), and `bus_type` ops marked `__ro_after_init` with kCFI-typed indirect dispatch; opcode is a `u8` constant from the spec command set, validated against per-helper allowlist.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-controller register/base disclosure behind CAP_SYSLOG; suppress `%p` in spmi tracepoints.
- **GRKERNSEC_DMESG** — restrict USID-out-of-range, missing-reg, and add-failure banners to CAP_SYSLOG so attackers cannot probe PMIC slave layout via dmesg.
- **R/W command allowlist** — framework helpers fan out to exactly one of seven opcodes (`SPMI_CMD_READ`, `SPMI_CMD_WRITE`, `SPMI_CMD_ZERO_WRITE`, `SPMI_CMD_EXT_READ`, `SPMI_CMD_EXT_WRITE`, `SPMI_CMD_EXT_READL`, `SPMI_CMD_EXT_WRITEL`) plus four state commands (RESET/SLEEP/WAKEUP/SHUTDOWN); no caller-supplied opcode reaches `ctrl->cmd` directly.
- **Slave-id PAX_USERCOPY** — `usid` parsed from DT `reg[0]` and bounded to `[0, SPMI_MAX_SLAVE_ID)` before assignment; per-device name (`%d-%02x`) derived strictly from controller `nr` + bounded `usid`.
- **Sysfs CAP_SYS_RAWIO** — any per-controller debug surface (e.g. raw PMIC register R/W exposed by master drivers) gated on CAP_SYS_RAWIO; the generic framework itself exposes no sysfs R/W of raw registers.
- **Master ops kCFI** — controller driver supplies `cmd/read_cmd/write_cmd` typed strictly per `struct spmi_controller`; kCFI rejects mismatched-signature dispatch.
- **Reset/sleep/wakeup/shutdown audit** — every state command emits a tracepoint and counts toward the bus tracepoint quota; rate-limited per controller via tracepoint policy.

Rationale: SPMI is the AP-to-PMIC control plane on most modern mobile SoCs. A length-bound miss in `spmi_ext_register_writel`, a missing `dev.type` check, or a free-without-quiesce on a controller can corrupt PMIC voltage rails, brick the device, or expose secure-world configuration. RAP/kCFI on master ops, command-opcode allowlisting, USID range enforcement, and refcount-overflow trapping convert the framework into a structural enforcement boundary over the PMIC attack surface.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `read_len_bound` | OOB | every helper rejects len==0 or len > spec-max (16 for ext, 8 for extl). |
| `short_addr_bound` | OOB | `spmi_register_read/_write` reject addr > 0x1F. |
| `usid_bound` | OOB | OF enumeration rejects USID ≥ SPMI_MAX_SLAVE_ID. |
| `ctrl_type_check` | TYPE | every cmd helper rejects ctrl whose `dev.type != &spmi_ctrl_type`. |

### Layer 2: TLA+

`models/spmi/probe_register.tla` (parent-declared): proves controller-add gated by `is_registered` interleaves correctly with `bus_register` so no slave device probe runs before the bus is live.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `spmi_drv_probe` post: pm_runtime is enabled iff probe returned 0; on failure the get/set_active/enable triple is fully unwound | `SpmiDriver::probe` |
| `spmi_controller_remove` post: all child slaves removed before `device_del` on controller | `Controller::remove` |
| `spmi_ext_register_writel(len)` precondition checked exactly once before `ctrl->write_cmd` dispatch | `SpmiDevice::ext_register_writel` |

### Layer 4: Verus/Creusot functional

Per-register R/W round-trip: `spmi_register_write(addr, v)` followed by `spmi_register_read(addr, &out)` yields `out == v` if no concurrent writer; encoded as Verus invariant chained with controller-driver write_cmd/read_cmd postconditions.

## Out of Scope

- Per-controller drivers (Qualcomm pmic-arb, MediaTek pmif, HiSilicon, Apple) — separate future Tier-3
- regmap-spmi (covered in `regmap.md` future Tier-3)
- 32-bit-only paths
- Implementation code
