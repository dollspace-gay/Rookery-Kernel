# Tier-3: drivers/slimbus/{core,messaging,sched}.c — SLIMbus framework (Serial Low-power Inter-chip Media Bus)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/slimbus/core.c
  - drivers/slimbus/messaging.c
  - drivers/slimbus/sched.c
  - drivers/slimbus/slimbus.h
  - include/linux/slimbus.h
-->

## Summary

The SLIMbus (Serial Low-power Inter-chip Media Bus, MIPI Alliance) framework — a multi-drop, multi-master, time-division-multiplexed serial bus designed for low-power audio/data interconnect between SoC and codec/PMIC peripherals on mobile platforms (Qualcomm SoCs, Apple SiP audio paths). Two-wire physical (clock + data), supports up to 16 active slave devices, message channel + per-stream isochronous data channels, with clock-gear scaling and clock-pause for power management.

This Tier-3 covers `core.c` (~547 lines: bus type, device alloc, logical address (LA) management, OF probe, device-present reporting), `messaging.c` (~365 lines: per-TID transaction submission, value/information-element read/write, slim_xfer_msg, slim_do_transfer, sanity checking), and `sched.c` (~121 lines: clock-pause reconfiguration sequence — `BEGIN_RECONFIGURATION` + `NEXT_PAUSE_CLOCK` + `RECONFIGURE_NOW` broadcasts).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct slim_controller` | per-bus controller (master) | `drivers::slimbus::Controller` |
| `struct slim_device` | per-slave device handle | `drivers::slimbus::SlimDevice` |
| `struct slim_driver` | client-driver registration | `drivers::slimbus::SlimDriver` |
| `struct slim_eaddr` | 6-byte enumeration address (manf_id + prod_code + dev_index + instance) | `drivers::slimbus::Eaddr` |
| `struct slim_msg_txn` | per-transaction descriptor (mt, mc, mc, ec, la, msg) | `drivers::slimbus::MsgTxn` |
| `struct slim_val_inf` | per-message value/info element (start_offset, num_bytes, rbuf/wbuf, comp) | `drivers::slimbus::ValInf` |
| `slim_register_controller(ctrl)` / `slim_unregister_controller(ctrl)` | controller bring-up/teardown | `Controller::register` / `_unregister` |
| `slim_device_report_present(ctrl, e_addr, *laddr)` | slave device-present REPORT_PRESENT response | `Controller::report_present` |
| `slim_report_absent(sbdev)` | drop slave from bus (LA invalidated) | `SlimDevice::report_absent` |
| `slim_get_logical_addr(sbdev)` | allocate/lookup per-device LA (0..15) | `SlimDevice::get_logical_addr` |
| `slim_do_transfer(ctrl, txn)` | submit messaging txn (incl. TID alloc + wait) | `Controller::do_transfer` |
| `slim_xfer_msg(sbdev, msg, mc)` | typed value/info-element transfer | `SlimDevice::xfer_msg` |
| `slim_read/_readb/_write/_writeb` | per-element R/W helpers | `SlimDevice::read*` / `write*` |
| `slim_msg_response(ctrl, reply, tid, len)` | controller-side TID response delivery | `Controller::msg_response` |
| `slim_alloc_txn_tid(ctrl, txn)` / `slim_free_txn_tid(ctrl, txn)` | per-TID id allocation (1..SLIM_MAX_TIDS) | `Controller::alloc_tid` / `_free_tid` |
| `slim_ctrl_clk_pause(ctrl, wakeup, restart)` | enter/exit clock-pause via reconfig broadcast | `Controller::clk_pause` |
| `__slim_driver_register(drv, owner)` / `slim_driver_unregister(drv)` | client driver register/unregister | `SlimDriver::register` / `_unregister` |
| `of_register_slim_devices(ctrl)` | DT-tree slave discovery (manf_id + prod_code from compatible) | `Controller::of_register_devices` |

## Compatibility contract

REQ-1: Bus topology — single controller (master) per bus, up to 16 slave devices addressed by 8-bit LA (LA 0xFF reserved for manager); each slave identified by 48-bit Enumeration Address (EA: manf_id:16 + prod_code:16 + dev_index:8 + instance:8).

REQ-2: Message channel framing — Message-Type (MT) + Message-Code (MC) + Element-Code (EC) + LA + payload (≤16 bytes for value/info elements per slicesize encoding); per-spec slicesize map { 1,2,3,4,6,8,12,16 } bytes.

REQ-3: Per-TID asynchronous transactions — REQUEST_VALUE / REQUEST_INFORMATION / REQUEST_CHANGE_VALUE / REQUEST_CLEAR_INFORMATION require Transaction-ID allocation (1..255) via `slim_alloc_txn_tid`; controller IRQ delivers per-TID response via `slim_msg_response`.

REQ-4: Per-element value/info-element addressing — 12-bit offset within element map; max num_bytes = 16; per-message sanity in `slim_val_inf_sanity` rejects offset+len > 0xC00 (3072 byte map bound).

REQ-5: Logical-address allocation — either `ctrl->get_laddr` (controller-managed, e.g. Qualcomm NGD) or generic `ida_alloc_max` (LA 0..14, SLIM_LA_MANAGER=0xFF excluded); LA assigned in REPORT_PRESENT.

REQ-6: Clock-pause sequence — `BEGIN_RECONFIGURATION` (broadcast) + `NEXT_PAUSE_CLOCK <restart>` + `RECONFIGURE_NOW`; refused (-EBUSY) if any TID still pending; wakeup path calls `ctrl->wakeup`.

REQ-7: Per-controller runtime-PM — `pm_runtime_get_sync(ctrl->dev)` for every transfer except clock-pause; transfers refused when `sched.clk_state != SLIM_CLK_ACTIVE` (except clock-pause sequence itself).

REQ-8: Device-bind matching — both OF (`of_driver_match_device`) and per-driver `slim_device_id` table (manf_id + prod_code + dev_index + instance must all match); modalias `slim:<dev_name>` with name `manf:prod:devidx:instance` hex.

REQ-9: Per-transfer timeout — caller-managed via `txn->rl + HZ` jiffies; on -ETIMEDOUT the TID is freed and a runtime-PM put is issued.

REQ-10: Manager-broadcast LA 0xFF (`SLIM_LA_MANAGER`) used for REPORT_PRESENT, ASSIGN_LA, and reconfiguration broadcasts; clients never use LA 0xFF for unicast.

REQ-11: Per-controller IDA ranges — global `ctrl_ida` (per-bus id), per-controller `laddr_ida` (per-LA pool, max 15 entries), per-controller `tid_idr` (per-TID, cyclic 1..SLIM_MAX_TIDS).

REQ-12: Per-stream port reservation tracked via `stream_list` per slim_device (cross-ref `slimbus-stream.md`); stream destruction must precede device removal.

## Acceptance Criteria

- [ ] AC-1: Probe Qualcomm SoC with Qualcomm NGD controller (`qcom-ngd-ctrl.c`); `dmesg | grep slimbus` shows bus registration + slave enumeration.
- [ ] AC-2: Codec slave (WCD9335 or WCD9340 on QCom platforms) gets LA assigned + value-element read/write succeeds.
- [ ] AC-3: Audio stream prepare/start/stop cycles through ports without TID leak (TID count returns to 0 in `/sys/kernel/debug/slimbus/`).
- [ ] AC-4: Clock-pause entry refused with -EBUSY when a transaction is pending; succeeds after drain.
- [ ] AC-5: Suspend/resume cycle leaves clock-pause state correctly restored; codec retains LA.
- [ ] AC-6: Malformed message (`num_bytes > 16` or offset out-of-range) returns -EINVAL.
- [ ] AC-7: Out-of-tree `slim_driver` module registers via `__slim_driver_register`; rejects bind without `id_table`/`of_match_table` or without `probe`.

## Architecture

`Controller` lives in `drivers::slimbus::Controller`:

```
struct Controller {
  dev: &Device,
  id: u32,                              // ida-allocated bus id
  name: KString,
  min_cg: u8, max_cg: u8,               // min/max clock gear
  laddr_ida: Ida,                       // 0..14 per-LA pool
  tid_idr: Idr<*mut MsgTxn>,            // 1..SLIM_MAX_TIDS per-TID
  txn_lock: SpinLock<()>,               // IRQ-safe TID idr lock
  lock: Mutex<()>,                      // per-controller mutex
  sched: Sched {
    clk_state: enum { ACTIVE, ENTERING_PAUSE, PAUSED, UNSPECIFIED },
    m_reconf: Mutex<()>,
    pause_comp: Completion,
  },
  xfer_msg:  fn(&Self, &MsgTxn) -> Result<()>,  // per-controller submit
  set_laddr: Option<fn(&Self, &Eaddr, u8) -> Result<()>>,
  get_laddr: Option<fn(&Self, &Eaddr, *mut u8) -> Result<()>>,
  wakeup:    Option<fn(&Self) -> Result<()>>,
}

struct SlimDevice {
  dev: Device,                          // bus = slimbus_bus, name = "manf:prod:dev:inst" hex
  e_addr: Eaddr,
  ctrl: Arc<Controller>,
  status: enum { DOWN, UP, RESERVED },
  laddr: u8,
  is_laddr_valid: bool,
  stream_list: List<StreamRuntime>,     // per-device stream registry
  stream_list_lock: SpinLock<()>,
}

struct MsgTxn {
  rl: u8, mt: u8, mc: u8, dt: u8, ec: u16,
  tid: u8, la: u8,
  msg: &mut ValInf,
  comp: Option<&Completion>,
}
```

Controller-init `slim_register_controller`:
1. Allocate bus id from global `ctrl_ida`.
2. Initialize laddr_ida + tid_idr + reconf-mutex + pause-completion.
3. Walk DT children: per OF child, parse `compatible` "slim<manf>,<prod>" + `reg = <devidx instance>`; alloc SlimDevice + register.

Per-message submit `slim_do_transfer(ctrl, txn)`:
1. If not clock-pause sequence: pm_runtime_get_sync(ctrl->dev); reject if clk_state != ACTIVE.
2. If MT/MC implies need_tid (per `slim_tid_txn`): alloc TID, attach completion (stack-allocated if msg->comp is NULL).
3. Call `ctrl->xfer_msg(ctrl, txn)`; on -ETIMEDOUT free TID.
4. If need_tid + no caller-completion: wait_for_completion_timeout(rl + HZ); on timeout free TID, return -ETIMEDOUT.
5. Drop pm_runtime if TX-only or on error.

Per-element transfer `slim_xfer_msg(sbdev, msg, mc)`:
1. `slim_val_inf_sanity` — reject num_bytes>16, offset+len > 0xC00, missing rbuf/wbuf per mc class.
2. Compute slicesize encoding from num_bytes.
3. Build EC = slicesize | (1<<3) | (offset<<4).
4. For CHANGE/CLEAR variants increase rl by num_bytes.
5. Dispatch via `slim_do_transfer`.

Clock-pause `slim_ctrl_clk_pause(ctrl, wakeup, restart)`:
- wakeup path: wait pause-completion(100ms), call `ctrl->wakeup`, mark clk_state ACTIVE.
- pause path: refuse with -EBUSY if any TID active in idr; broadcast BEGIN_RECONFIGURATION + NEXT_PAUSE_CLOCK + RECONFIGURE_NOW; on success mark PAUSED + complete pause-completion.

## Hardening

(Inherits row-1 features from `drivers/bus/00-overview.md` § Hardening.)

slimbus-core specific reinforcement:

- **`slim_val_inf_sanity` rejects oversize/oob messages** before per-controller xfer_msg dispatch.
- **TID idr cyclic alloc bounded by `SLIM_MAX_TIDS`** with GFP_ATOMIC under txn_lock; alloc failure returns to caller.
- **`txn_lock` is IRQ-safe spinlock** — required because controller IRQ delivers per-TID responses via `slim_msg_response`.
- **REPORT_PRESENT rejected when clk_state != ACTIVE** — defense against stale enumeration over a paused bus.
- **Per-driver `id_table` AND `probe` required** at `__slim_driver_register` — refuses driver without bind discriminator.
- **LA range bounded to [0, SLIM_LA_MANAGER-1]** in `ida_alloc_max`; manager LA 0xFF never assigned to slaves.
- **Per-transfer pm_runtime_put on every error path** including TID free-on-timeout — defense against runtime-PM refcount leak.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `slim_device`, `slim_controller`, and per-transaction `slim_msg_txn`/`slim_val_inf` buffers; `rbuf`/`wbuf` are bounded ≤16 bytes per spec and never directly copied to/from userspace from the framework.
- **PAX_KERNEXEC** — SLIMbus framework core in W^X kernel text; `bus_type`, `slim_driver` vtables, and controller `xfer_msg`/`wakeup`/`get_laddr`/`set_laddr` callbacks live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `slim_do_transfer`, `slim_xfer_msg`, `slim_msg_response` (IRQ path), and `slim_ctrl_clk_pause`.
- **PAX_REFCOUNT** — saturating `refcount_t` on `slim_controller` (per-bus pin) and per-`slim_device` `dev` refcount; pm_runtime get/put paired strictly across all error branches; overflow trap defeats attach/detach race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `slim_device`/`slim_msg_txn`/`slim_val_inf` slab objects so stale enumeration addresses, response payloads, and TID state never bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every controller-driver entry; SLIMbus framework itself owns no userland surface, but the helper buffers it passes to controller drivers are never user pointers.
- **PAX_RAP / kCFI** — `slim_driver` vtable (`probe`/`remove`/`shutdown`/`device_status`), `slim_controller` callback table (`xfer_msg`/`wakeup`/`get_laddr`/`set_laddr`), and bus_type ops marked `__ro_after_init` with kCFI-typed indirect dispatch; async-response completion path entered only via the registered controller IRQ vector.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-controller register/base pointer disclosure behind CAP_SYSLOG; suppress `%p` in transfer/response tracepoints.
- **GRKERNSEC_DMESG** — restrict transfer-timeout, sanity-check-fail, and LA-exhaustion banners to CAP_SYSLOG so attackers cannot probe codec presence/state via dmesg.
- **EA-array bounds at OF probe** — `manf_id`/`prod_code` parsed via `sscanf("slim%x,%x")` rejected if not exactly two tokens; `reg` array required to be exactly 2 cells.
- **Async-callback RAP** — `slim_driver::device_status` and `slim_val_inf::comp` invoked only through kCFI-typed dispatch; completion pointer validated non-NULL before `complete()` in IRQ path.
- **Controller PAX_REFCOUNT** — `slim_controller::id` released via `ida_free` on unregister; per-device `is_laddr_valid` cleared under `ctrl->lock` before `ida_free(laddr_ida, laddr)`, defeating dangling-LA reuse.
- **Scheduler lock IRQ-safe** — `m_reconf` is a process-context mutex but `txn_lock` is IRQ-safe spinlock; clock-pause refuses if any TID is pending under `txn_lock`, defeating a paused-with-pending-response stall.
- **TID id-space hardening** — `tid_idr` cyclic allocation never re-issues a live TID; `slim_msg_response` validates `tid` before dereferencing `txn->msg->rbuf`.
- **CAP_SYS_RAWIO** on any out-of-tree controller debugfs surface; framework itself exposes no debugfs.

Rationale: SLIMbus connects the SoC to security-relevant peripherals (codecs, PMIC sub-blocks). A TID-alloc race, an unbounded `num_bytes`, or a missing IRQ-safe spinlock on the response path can leak codec state, hang the bus, or corrupt audio DMA buffers belonging to other clients. RAP/kCFI on `xfer_msg`/`device_status`, refcount-overflow trapping on controller/device pins, and message-sanity gating turn the framework from "trust the controller driver" into a structural enforcement boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `val_inf_no_oob` | OOB | `slim_val_inf_sanity` rejects every msg with `num_bytes > 16` or `offset + num_bytes > 0xC00`. |
| `tid_no_collision` | UNIQUENESS | `tid_idr` cyclic allocator never returns a TID already mapped to a live `MsgTxn`. |
| `laddr_no_collision` | UNIQUENESS | per-controller `laddr_ida` allocator bounded to [0, SLIM_LA_MANAGER-1]; never re-issues active LA. |
| `pause_no_pending` | LIVENESS | clock-pause refuses entry while any TID is pending; no transfer can be submitted while clk_state==PAUSED. |

### Layer 2: TLA+

`models/slimbus/clk_pause.tla` (parent-declared): proves the three-broadcast clock-pause reconfiguration sequence (BEGIN + NEXT_PAUSE + RECONFIGURE_NOW) observes the spec-required ordering across concurrent transmit attempts; pause/wakeup completion handshake serializes correctly.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `slim_do_transfer` post: every `-ETIMEDOUT` path frees the TID and drops pm_runtime | `Controller::do_transfer` |
| `slim_msg_response` post: TID is freed exactly once; `rbuf` writes bounded by `len ≤ 16` | `Controller::msg_response` |
| `slim_ctrl_clk_pause(false)` post: clk_state ∈ {ACTIVE, PAUSED} on success; ENTERING_PAUSE only transient inside reconf-mutex | `Controller::clk_pause` |

### Layer 4: Verus/Creusot functional

Per-message round-trip: `slim_xfer_msg` → `slim_do_transfer` → controller IRQ → `slim_msg_response` → caller completion observed; encoded as Verus invariant chained with controller-driver `xfer_msg` postcondition.

## Out of Scope

- Per-controller drivers (`qcom-ngd-ctrl.c`) — separate Tier-3 future
- SLIMbus stream/port runtime (`stream.c`) — covered in `slimbus-stream.md` future Tier-3
- 32-bit-only paths
- Implementation code
