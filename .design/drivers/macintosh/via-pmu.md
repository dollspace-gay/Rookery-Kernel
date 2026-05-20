# Tier-3: drivers/macintosh/{via-pmu,via-pmu-event}.c — PowerMac VIA-PMU driver (power-management micro, RTC, ADB, /dev/pmu)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/macintosh/via-pmu.c
  - drivers/macintosh/via-pmu-event.c
  - drivers/macintosh/via-pmu-event.h
  - include/uapi/linux/pmu.h
  - include/linux/adb.h
-->

## Summary

The VIA-PMU (Power Management Unit) is the on-board Motorola micro-controller bridged to the host CPU via a 6522 VIA on PowerMacs from the PowerBook 3400 era through the G4 Mini and original iBook line. It owns power-rail sequencing (sleep/wake, hard-power-off, hard-drive spin-down), the real-time clock, the ADB (Apple Desktop Bus) keyboard/mouse controller, battery monitoring, environment sensors (lid switch, AC plug, brightness up/down keys), and backlight control. On every PowerMac that uses it, the kernel must drive the PMU to suspend correctly, set the wall clock, and route ADB input. The PMU is *the* gatekeeper between Linux/PPC and the platform.

The driver registers itself as the ADB-bus driver (`via_pmu_driver`) so all ADB transactions flow through `pmu_send_request`. It also publishes a `/dev/pmu` misc-style char device (major 10, minor `PMU_MINOR = 154` historically; now misc-class) — userland (`pmud`, `pbbuttonsd`) opens it to receive button-event notifications (lid close, power button, sound brightness keys) and to issue `PMU_IOC_{SLEEP,CAN_SLEEP,GET_BACKLIGHT,SET_BACKLIGHT,GET_MODEL,HAS_ADB,GRAB_BACKLIGHT}` ioctls. The driver also exports `/proc/pmu/` entries (`info`, `irqstats`, `options`, `battery_N`) for legacy userspace.

This Tier-3 covers `via-pmu.c` (~2670 lines: PMU state machine, command FIFO, ADB protocol, suspend/resume, /dev/pmu, /proc/pmu) and the much smaller `via-pmu-event.c` (~80 lines: lid/power-button event injection into the input subsystem).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `via_pmu_driver` (`struct adb_driver`) | ADB-bus driver registration | `drivers::macintosh::pmu::AdbDriver` |
| `find_via_pmu()` | OF/DT probe + per-chip kind detection | `Subsystem::probe` |
| `pmu_init()` | post-probe init, IRQ register, interrupt-mask program | `Subsystem::init` |
| `pmu_request(req, done, nbytes, ...)` / `pmu_send_request(req, sync)` | issue a PMU command | `Pmu::request` / `_send_request` |
| `pmu_queue_request(req)` | enqueue async PMU command | `Pmu::queue_request` |
| `pmu_poll()` / `pmu_poll_adb()` | poll-mode (early boot / no IRQ) command dispatch | `Pmu::poll` / `_poll_adb` |
| `via_pmu_interrupt(irq, arg)` | IRQ handler — drain RX FIFO, deliver replies/intr-events | `Pmu::interrupt` |
| `pmu_pass_intr(data, len)` | broadcast PMU-INT event payload to all `/dev/pmu` openers | `DevPmu::pass_intr` |
| `pmu_open` / `pmu_read` / `pmu_release` / `pmu_fpoll` / `pmu_unlocked_ioctl` / `compat_pmu_ioctl` | `/dev/pmu` fops | `DevPmu::file_ops` |
| `pmu_ioctl(filp, cmd, arg)` | ioctl dispatch (sleep, model, has-adb, backlight, can-sleep, grab-backlight) | `DevPmu::ioctl` |
| `pmu_present()` | predicate — is the PMU initialized | `Pmu::present` |
| `pmu_set_rtc_time(tm)` / `pmu_get_rtc_time(tm)` | RTC read/write | `Pmu::rtc_set` / `_get` |
| `pmu_shutdown()` / `pmu_restart()` | platform power-off / restart | `Pmu::shutdown` / `_restart` |
| `powerbook_sleep()` / `pmac_suspend_disable_irqs()` / `pmac_suspend_enable_irqs()` | platform-suspend integration | `Pmu::suspend_*` |
| `pmu_pm_ops` (`struct platform_suspend_ops`) | S3 entry/exit | `Pmu::PmOps` |
| `pmu_batteries[]`, `pmu_power_flags`, `pmu_battery_count` | battery monitor state | `Pmu::battery_*` |
| `via_pmu_event_init()` (in `via-pmu-event.c`) | register input device for lid/power button | `PmuEvent::init` |
| `via_pmu_event(event, value)` | inject input event from PMU IRQ | `PmuEvent::inject` |
| `via_pmu_proc_init()` | install `/proc/pmu/{info,irqstats,options,battery_N}` | `Pmu::proc_init` |

## Compatibility contract

REQ-1: ABI `include/uapi/linux/pmu.h` UAPI must remain stable: ioctl numbers `PMU_IOC_{SLEEP,CAN_SLEEP,GET_BACKLIGHT,SET_BACKLIGHT,GET_MODEL,HAS_ADB,GRAB_BACKLIGHT}` and the `PMU_*` command bytes (`PMU_SLEEP`, `PMU_SHUTDOWN`, `PMU_SET_RTC`, `PMU_READ_RTC`, `PMU_POWER_EVENTS`, `PMU_ADB_CMD`, `PMU_SET_INTR_MASK`, `PMU_SYSTEM_READY`, `PMU_RESET`, `PMU_POWER_CTRL`, `PMU_POWER_CTRL0`).

REQ-2: PMU kind detection at probe — `PMU_OHARE_BASED`, `PMU_HEATHROW_BASED`, `PMU_PADDINGTON_BASED`, `PMU_KEYLARGO_BASED` from OpenFirmware compatible string + parent device; `pmu_intr_mask` and feature gates derived per-kind.

REQ-3: ADB-bus integration — `via_pmu_driver` implements `probe`, `init`, `send_request`, `autopoll`, `poll`, `reset_bus` of `struct adb_driver`; ADB packets travel via `PMU_ADB_CMD` PMU command.

REQ-4: PMU command FIFO model — at most one outstanding command at a time (`pmu_state` ∈ `idle / sending / intack / writing / sleeping / awaking`); `pmu_lock` (spinlock) protects state machine + cmd queue.

REQ-5: RX delivery model — IRQ handler reads the response into `current_req` buffer, calls `req->done(req)`; `PMU_INT_*` interrupt events (PCEJECT, SNDBRT, ADB, TICK, ENVIRONMENT) are fanned out via `pmu_pass_intr` to every open `/dev/pmu` file's per-fd ring buffer (`RB_SIZE = 16` entries).

REQ-6: `/dev/pmu` read returns one PMU-INT event at a time (variable length up to 16 bytes); `O_NONBLOCK` honoured; `poll` reports `POLLIN` when ring non-empty.

REQ-7: `/dev/pmu` write is a no-op (returns 0) — userland cannot inject PMU commands directly.

REQ-8: `PMU_IOC_SLEEP` requires CAP_SYS_ADMIN; calls `pm_suspend(PM_SUSPEND_MEM)` which invokes `pmu_pm_ops.enter = powerbook_sleep`.

REQ-9: Battery polling every `BATTERY_POLLING_COUNT` PMU ticks; `pmu_batteries[]` exposed via `/proc/pmu/battery_N` text format.

REQ-10: Lid-switch + power-button events injected into the input subsystem as `KEY_POWER` / `SW_LID` via `via_pmu_event`; consumers (logind, acpid-equivalent) react.

REQ-11: `/proc/pmu/options` `lid_wakeup` toggle requires CAP_SYS_ADMIN.

## Acceptance Criteria

- [ ] AC-1: Boot on a PowerBook G3 Wallstreet (PMU_PADDINGTON_BASED) or PowerBook G4 (PMU_KEYLARGO_BASED); `/dev/pmu` and `/proc/pmu/` present.
- [ ] AC-2: `pmud`/`pbbuttonsd` userland: lid close generates an event readable from `/dev/pmu`.
- [ ] AC-3: `hwclock --systohc` updates RTC via `pmu_set_rtc_time`; subsequent `hwclock` round-trips.
- [ ] AC-4: `echo mem > /sys/power/state` (with CAP_SYS_ADMIN) enters sleep, wake on lid/power resumes; no irrecoverable ADB state loss.
- [ ] AC-5: ADB keyboard input continues to function after suspend/resume cycle.
- [ ] AC-6: Battery `/proc/pmu/battery_0` reports charge/voltage/current; values plausible on real HW.
- [ ] AC-7: `PMU_IOC_SLEEP` from non-root returns `-EACCES`; from root suspends.
- [ ] AC-8: Force-poweroff via `shutdown -h now` invokes `pmu_shutdown`; system powers off (observed externally).

## Architecture

`Pmu` lives in `drivers::macintosh::pmu::Pmu`:

```
struct Pmu {
  via: NonNull<u8>,            // VIA 6522 MMIO base (memremap'd from OF)
  irq: u32,
  gpio_irq: Option<u32>,
  kind: PmuKind,               // OHARE / HEATHROW / PADDINGTON / KEYLARGO / UNKNOWN
  version: u8,
  has_adb: bool,
  intr_mask: u8,
  state: AtomicU32,            // idle / sending / intack / writing / sleeping / awaking
  current_req: Mutex<Option<KBox<AdbRequest>>>,
  cmd_queue: SpinLock<VecDeque<KBox<AdbRequest>>>,
  fully_inited: AtomicBool,
  suspended: AtomicBool,
  irq_stats: [AtomicU32; NUM_IRQ_STATS],
  proc_root: Option<KBox<ProcDir>>,
  batteries: Mutex<[BatteryInfo; PMU_MAX_BATTERIES]>,
  power_flags: AtomicU32,       // PMU_PWR_AC_PRESENT etc.
}

struct PmuPrivate {                // per-/dev/pmu fd
  list: ListLinks,                 // all_pmu_pvt
  rb_get: u8,
  rb_put: u8,
  rb_buf: [RbEntry; 16],
  wait: WaitQueue,
  lock: SpinLock<()>,
  backlight_locker: bool,
}
```

PMU command dispatch `pmu_request(req, done, nbytes, cmd, ...)`:
1. Validate `nbytes <= 16`; reject `cmd >= PMU_MAX_CMDS`.
2. Populate `req->data[0..nbytes]`; `req->reply_expected` per `pmu_data_len[cmd][1] != 0`.
3. `pmu_queue_request(req)` — append under `pmu_lock`.
4. `pmu_start()` — if state == idle, write VIA register `vIER` to assert SR-out, transition state → sending.
5. IRQ handler advances state machine: sends bytes from `req->data`, reads response bytes into `req->reply`.
6. On completion: `req->complete = 1`, `req->done(req)` callback fires, queue advanced.

ADB integration: `pmu_send_request(req, sync)` wraps `pmu_request` with cmd `PMU_ADB_CMD`; for `sync`, polls until `req->complete`.

IRQ event delivery: when PMU sends a `PMU_INT_*` notification (not in response to our cmd), the IRQ handler reads `intr_data`, then calls `pmu_pass_intr(data, len)` which iterates `all_pmu_pvt` under `all_pvt_lock` and per-fd advances `rb_put` (drop on overflow), waking blocked readers.

`/dev/pmu` open: alloc `pmu_private`, init ring buf + wait queue, add to `all_pmu_pvt`, install as `file->private_data`.
`/dev/pmu` read: pop one event from ring buf into user buf via `copy_to_user`; bounded by `count` and per-entry length.
`/dev/pmu` ioctl: dispatch on `cmd` — `PMU_IOC_SLEEP` (CAP_SYS_ADMIN, `pm_suspend(MEM)`), `PMU_IOC_CAN_SLEEP`, `PMU_IOC_{GET,SET,GRAB}_BACKLIGHT`, `PMU_IOC_{GET_MODEL,HAS_ADB}` (read-only).

Suspend path `powerbook_sleep`:
1. Wait for outstanding PMU requests (`batt_req.complete`).
2. `enable_kernel_fp()` / `enable_kernel_altivec()`.
3. Per-kind: `powerbook_sleep_3400` / `_grackle` / `_Core99`.
4. PMU command `PMU_SLEEP 'M' 'A' 'T' 'T'` issued, CPU halts.
5. Wake IRQ fires; `pmac_suspend_enable_irqs` re-arms ADB poll.

RTC: `pmu_get_rtc_time` issues `PMU_READ_RTC` (4-byte big-endian seconds since Mac epoch 1904-01-01); `pmu_set_rtc_time` issues `PMU_SET_RTC` with converted seconds.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cmd_nbytes_oob` | OOB | `pmu_request` rejects `nbytes > 16`; `req.data` buffer access bounded. |
| `rb_buf_oob` | OOB | per-fd ring buffer indices bounded by `RB_SIZE = 16`; wrap is correct. |
| `state_machine_invariant` | RACE | `pmu_state` transitions only via the defined edges; no state-machine bypass. |
| `pmu_private_no_uaf` | UAF | per-fd `pmu_private` removed from `all_pmu_pvt` before `kfree`; IRQ `pass_intr` cannot race. |

### Layer 2: TLA+

`models/macintosh/pmu_fsm.tla` — proves the `idle ↔ sending ↔ intack ↔ writing` state machine with concurrent `pmu_queue_request` + IRQ delivery cannot deadlock or duplicate a request.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `pmu_request` post: req queued iff `pmu_state` advances at least one byte | `Pmu::request` |
| `pmu_pass_intr` post: every open fd receives the event or experiences a defined drop | `DevPmu::pass_intr` |
| `pmu_ioctl(PMU_IOC_SLEEP)` precondition: `capable(CAP_SYS_ADMIN)` | `DevPmu::ioctl` |
| `pmu_get_rtc_time` ↔ `pmu_set_rtc_time` round-trip preserves Mac-epoch seconds | `Pmu::rtc_*` |

### Layer 4: Verus functional

Sequence of `pmu_queue_request` events with concurrent IRQ delivery produces request completion order equal to enqueue order (FIFO); RX events broadcast to all open fds preserve per-fd local FIFO order. Encoded as Verus refinement over the cmd queue + per-fd ring transcripts.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

via-pmu specific reinforcement:

- **Per-fd ring buffer fixed-size** (`RB_SIZE = 16`) — defense against per-fd unbounded queueing under PMU-INT storm.
- **`pmu_lock` spinlock IRQ-safe** — defense against IRQ-vs-syscall state-machine corruption.
- **PMU cmd `nbytes` bounded by 16** — defense against caller passing oversize data.
- **`pmu_data_len` table validates per-cmd RX/TX length** — defense against arbitrary cmd byte values.
- **/proc/pmu mutex (`pmu_info_proc_mutex`) serializes ioctl + read paths** — defense against race-on-suspend.
- **`pmu_present()` predicate** — every external entry first checks PMU is fully initialised; defense against pre-init misuse.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `pmu_private`, `pmu_battery_info`, and per-fd ring-buffer entries; `pmu_read` `copy_to_user` strictly bounded by `rb_entry.len <= 16`.
- **PAX_KERNEXEC** — via-pmu driver text + `pmu_device_fops`, `via_pmu_driver`, and `pmu_pm_ops` in W^X kernel text; `__ro_after_init` for `pmu_data_len[]` cmd table and per-kind feature flag tables.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `pmu_unlocked_ioctl`, `pmu_request`, `via_pmu_interrupt`, `powerbook_sleep`, and `pmu_get_rtc_time`/`_set_rtc_time`.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-fd `pmu_private` (via fd table) and `current_req`/`batt_req` `adb_request` `complete` flags; overflow trap defeats IRQ-vs-release race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `pmu_private` rb_buf + cmd/response buffers on close, battery state on disconnect, and `current_req` on completion so stale PMU payloads and RTC bytes cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every `/dev/pmu` read/ioctl entry; `copy_to_user`/`get_user`/`put_user` paths validated.
- **PAX_RAP / kCFI** — `pmu_device_fops`, `via_pmu_driver` adb-ops, `pmu_pm_ops` suspend ops, and per-cmd done callbacks (`req->done`) marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-VIA MMIO pointer disclosure behind CAP_SYSLOG; suppress `%p` in PMU-INT tracepoints and `/proc/pmu/info` raw-pointer fields.
- **GRKERNSEC_DMESG** — restrict PMU init banners, ADB reset banners, suspend/resume diagnostics, and battery-state-fail banners to CAP_SYSLOG so attackers cannot fingerprint PMU revision via dmesg.
- **/dev/pmu CAP_SYS_RAWIO open** — opening `/dev/pmu` requires CAP_SYS_RAWIO in the user namespace; refuse open under lockdown integrity tier.
- **`PMU_*` cmd allowlist** — direct PMU command injection from `/dev/pmu` write is rejected (write returns 0); all PMU commands flow only from in-kernel call sites against `pmu_data_len[]`.
- **RTC write CAP_SYS_TIME** — `pmu_set_rtc_time` (and the rtc-class wrapper that calls it) require CAP_SYS_TIME; defense against unprivileged time skew.
- **Sleep-state CAP_SYS_ADMIN** — `PMU_IOC_SLEEP` already enforces CAP_SYS_ADMIN; lid-wakeup option toggle in `/proc/pmu/options` also CAP_SYS_ADMIN.
- **Battery info read** — `/proc/pmu/battery_N` and `pmu_battery_info` user-readable; PAX_USERCOPY caps copy size to `sizeof(pmu_battery_info)`.
- **Backlight grab single-locker** — `PMU_IOC_GRAB_BACKLIGHT` keyed on per-fd `backlight_locker`; on release, backlight re-enabled exactly once.

Rationale: the PMU on PowerMacs is the platform-control hardware path — incorrect handling of its command FIFO, IRQ delivery, or `/dev/pmu` UAPI can corrupt the RTC, hang ADB input, or trigger uncoordinated S3 entry. CAP_SYS_RAWIO on `/dev/pmu`, CAP_SYS_ADMIN on sleep, CAP_SYS_TIME on RTC writes, RAP/kCFI on adb-driver/file ops, refcount-overflow trapping on request/private structs, and bounded ring buffers turn via-pmu from "open the node, control the laptop" into a structurally bounded service interface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VIA-CUDA driver (`via-cuda.c`) — older PMU equivalent, covered separately.
- macio-asic + macio-adb bus glue (`macio_asic.c`, `macio-adb.c`) — covered separately.
- SMU (System Management Unit, newer G5) driver (`smu.c`) — separate Tier-3.
- Windfarm thermal control (`windfarm_*.c`) — separate Tier-3.
- ADBHID input bridge (`adbhid.c`) — separate Tier-3.
- 32-bit-only paths.
- Implementation code.
