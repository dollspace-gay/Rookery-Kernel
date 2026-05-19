# Tier-3: drivers/ptp/{ptp_clock,ptp_chardev,ptp_sysfs}.c — PTP hardware-clock framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/ptp/00-overview.md
upstream-paths:
  - drivers/ptp/ptp_clock.c
  - drivers/ptp/ptp_chardev.c
  - drivers/ptp/ptp_sysfs.c
  - drivers/ptp/ptp_vclock.c
  - drivers/ptp/ptp_private.h
  - include/linux/ptp_clock_kernel.h
  - include/uapi/linux/ptp_clock.h
-->

## Summary

The PTP (IEEE 1588) hardware-clock framework abstracts per-device PHC (PTP Hardware Clock) silicon — NIC MAC clocks (Intel `ice`/`igc`/`igb`/`ixgbe`, Mellanox `mlx5`, Marvell `octeon`), discrete time-sync ASICs (`ptp_ocp` Open Compute Time Card, IDT 82P33/ClockMatrix/FC3, Renesas RZ), virtualization-host clocks (`ptp_kvm`, `ptp_vmw`, `ptp_vmclock` for confidential VMs), VLAN-domain virtual-PHCs (`ptp_vclock`), and software-pulled platform clocks (`ptp_dfl_tod`, `ptp_mock`). Each driver registers a `struct ptp_clock_info` callback table; the framework synthesizes a `/dev/ptpN` character device backed by `posix-clock`, exposes sysfs (`/sys/class/ptp/ptpN/`), supports external timestamping (extts), periodic-output generation (perout), PPS callbacks, programmable-pin function multiplexing, system↔device cross-timestamp ioctls, and phase adjustment. Userspace tools (`ptp4l` / `phc2sys` / `ts2phc` / `linuxptp`) drive frequency / offset / phase tuning via the POSIX clock interface and PTP-specific ioctls.

This Tier-3 covers `ptp_clock.c` (~710 lines: posix-clock ops bridge, `ptp_clock_register/unregister`, event queue, vclock fan-out), `ptp_chardev.c` (~640 lines: ioctl dispatch, per-fd timestamp-event queue, pin-function configuration), `ptp_sysfs.c` (~480 lines: per-cap sysfs attributes + extts/perout/pps enable + per-pin sysfs group).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ptp_clock` | runtime per-PHC state (posix_clock, info, queues, sysfs, vclocks) | `drivers::ptp::PtpClock` |
| `struct ptp_clock_info` | per-driver const ops (gettime64/gettimex64, settime64, adjtime, adjfine, adjphase, getmaxphase, enable, verify, getcrosststamp, n_alarm, n_ext_ts, n_per_out, n_pins, pin_config[]) | `drivers::ptp::ClockInfo` |
| `struct posix_clock_operations ptp_clock_ops` | POSIX-clock back-end (gettime/settime/getres/adjtime/ioctl/open/release/poll/read) | `drivers::ptp::PosixOps` |
| `ptp_clock_register(info, parent)` / `ptp_clock_unregister(ptp)` | per-driver lifecycle | `PtpClock::register` / `_unregister` |
| `ptp_clock_index(ptp)` / `_by_of_node(np)` / `_by_dev(dev)` | per-PHC index lookups (used by NIC drivers + linuxptp) | `PtpClock::index` / `_by_of_node` / `_by_dev` |
| `ptp_clock_event(ptp, *event)` | driver→core event push (EXTTS / PPS / PPSUSR / EXTOFF) | `PtpClock::event` |
| `ptp_clock_freerun(ptp)` | true if any vclock is bound (freezes settime / adjtime on parent) | `PtpClock::freerun` |
| `ptp_open(pccontext, fmode)` / `ptp_release(pccontext)` / `ptp_read(...)` / `ptp_poll(...)` / `ptp_ioctl(pccontext, cmd, arg)` | char-device ops | `PtpChardev::open` / `_release` / `_read` / `_poll` / `_ioctl` |
| `ptp_disable_all_events(ptp)` / `ptp_set_pinfunc(ptp, pin, func, chan)` / `ptp_disable_pinfunc(info, func, chan)` | pin / event mux | `PtpChardev::disable_all_events` / `_set_pinfunc` / `_disable_pinfunc` |
| `ptp_find_pin(ptp, func, chan)` / `_unlocked` | DT helper for assigning a pin to a function | `PtpClock::find_pin` |
| `ptp_schedule_worker(ptp, delay)` / `_cancel_worker_sync(ptp)` | per-PHC kthread-worker | `PtpClock::schedule_worker` / `_cancel_worker_sync` |
| `struct ptp_clock_request` / `ptp_clock_event` / `ptp_extts_event` | event + request structs | `drivers::ptp::ClockRequest` / `ClockEvent` / `ExttsEvent` |
| `ptp_vclock_register(pclock)` / `_unregister` | virtual-clock (VLAN-domain) variant | `drivers::ptp::Vclock` |
| sysfs attrs `clock_name`, `max_adjustment`, `n_alarms`, `n_external_timestamps`, `n_periodic_outputs`, `n_programmable_pins`, `pps_available`, `extts_enable`, `fifo`, `period`, `pps_enable`, `n_vclocks`, `max_vclocks`, `max_phase_adjustment` | per-cap surface | `PtpSysfs::*` |

## Compatibility contract

REQ-1: per-driver registers via `ptp_clock_register(info, parent)`; framework allocates a `posix_clock`, registers it via `posix_clock_register` with major `MISC_MAJOR`-derived device number; result `/dev/ptpN` cdev + `/sys/class/ptp/ptpN/`.

REQ-2: per-`ptp_clock_info` callbacks must implement at least `gettime64` (or `gettimex64`), `settime64`, `adjtime`, `adjfine`, and `enable`; others optional. `getmaxphase`+`adjphase` together advertise phase adjustment.

REQ-3: per-fd event queue (`struct timestamp_event_queue`) ring of `PTP_MAX_TIMESTAMPS = 128` events; per-channel mask (`PTP_MAX_CHANNELS = 16` default) selects which extts channels deposit events; per-fd bitmap allocated at open.

REQ-4: ioctl set on `PTP_CLK_MAGIC = '='`: `PTP_CLOCK_GETCAPS{,2}`, `PTP_EXTTS_REQUEST{,2}`, `PTP_PEROUT_REQUEST{,2}`, `PTP_ENABLE_PPS{,2}`, `PTP_SYS_OFFSET{,2}`, `PTP_PIN_GETFUNC{,2}`, `PTP_PIN_SETFUNC{,2}`, `PTP_SYS_OFFSET_PRECISE{,2,_CYCLES}`, `PTP_SYS_OFFSET_EXTENDED{,2,_CYCLES}`, `PTP_MASK_CLEAR_ALL`, `PTP_MASK_EN_SINGLE`.

REQ-5: POSIX clock dispatch: `ptp_clock_settime` rejects if any vclock is bound (`ptp_clock_freerun`); `ptp_clock_adjtime` decodes `ADJ_SETOFFSET`/`ADJ_FREQUENCY`/`ADJ_OFFSET` and dispatches to `adjtime`/`adjfine`/`adjphase`; `tx->freq` validated against `info->max_adj` (ppb).

REQ-6: per-pin function map: `n_pins` per device with `ptp_pin_desc[]`; functions `PTP_PF_NONE`/`_EXTTS`/`_PEROUT`/`_PHYSYNC`; setfunc enforces driver `verify(info, pin, func, chan)` callback; mutually-exclusive channel/function relocation handled via `disable_pinfunc` of the prior owner.

REQ-7: per-`enable`-call serialized via `ptp->pincfg_mux` mutex; driver `enable(info, req, on)` is the single touchpoint for arming / disarming a feature; idempotent for `on==0`.

REQ-8: extts event delivery: driver pushes via `ptp_clock_event(ptp, ev)`; framework fans out to every per-fd queue whose channel-mask bit `ev->index` is set; queue overflow advances `head` (drops oldest).

REQ-9: cross-timestamping: `PTP_SYS_OFFSET_PRECISE` / `_EXTENDED` use `getcrosststamp`/`gettimex64` to bracket PHC reads with TSC/ART for sub-µs synchronization; `_CYCLES` variants return raw cycles instead of post-conversion ns.

REQ-10: PPS integration: if `info->pps == 1`, PHC drives a software-visible PPS via `pps_kernel`; `PPS_CAPTUREASSERT | PPS_OFFSETASSERT | PPS_CANWAIT | PPS_TSFMT_TSPEC` modes.

REQ-11: vclock fan-out (`ptp_vclock_register`): per-VLAN virtual PHC chained off a physical PHC; `n_vclocks` sysfs gate; physical clock enters "freerun" once any vclock is alive (cannot settime / adjtime parent independently).

REQ-12: each `ptp_clock_event` `EXTOFF` carries `offset` (ns) rather than absolute timestamp; encoded with `PTP_EXT_OFFSET` flag in delivered `ptp_extts_event`.

## Acceptance Criteria

- [ ] AC-1: `ls /sys/class/ptp/` lists each NIC PHC + each discrete time-card; `cat /sys/class/ptp/ptp0/clock_name` matches the driver's `info->name`.
- [ ] AC-2: `ptp4l -i eth0 -m -2 -P -s` brings `eth0` into PHC slave mode against a master on the wire; clock offset < 1 µs sustained.
- [ ] AC-3: `phc2sys -s ptp0 -c CLOCK_REALTIME -O 0 -m` reflects PHC time to `CLOCK_REALTIME`; round-trip via `clock_gettime(CLOCK_TAI)` consistent.
- [ ] AC-4: extts: program a pin to EXTTS, drive a 1 PPS into it from a signal generator, `linuxptp/testptp -e 1` captures events with stable inter-arrival.
- [ ] AC-5: perout: program a 1 Hz PEROUT, scope-verify edges align with `gettime64`.
- [ ] AC-6: PTP_SYS_OFFSET_PRECISE: returns ART/TSC + PHC pair on TSC-locked Intel NIC; jitter < 50 ns.
- [ ] AC-7: vclock: bind 2 vclocks on `eth0`; parent enters freerun (`/sys/class/ptp/ptp0/n_vclocks > 0` and settime returns `-EBUSY`); each vclock independently sync-able.
- [ ] AC-8: pin mux: re-assign a previously-EXTTS pin to PEROUT via `PTP_PIN_SETFUNC2`; prior EXTTS subscription deactivated; no event leakage.
- [ ] AC-9: PPS: enable PPS on a PHC declaring `pps==1`; `/dev/pps0` consumer sees stable assertions.

## Architecture

`PtpClock` lives in `drivers::ptp::PtpClock`:

```
struct PtpClock {
  clock: PosixClock,                   // posix_clock-registered
  dev: Device,                         // class = ptp_class, name = "ptpN"
  info: &'static ClockInfo,            // per-driver const ops
  index: u32,                          // matches /dev/ptpN
  dialed_frequency: i64,
  tsevqs: Mutex<LinkedList<TimestampEventQueue>>, // per-fd queues
  tsevqs_lock: SpinLock,
  pincfg_mux: Mutex<()>,
  is_virtual_clock: bool,
  has_cycles: bool,
  defunct: bool,
  n_vclocks: AtomicU32,
  max_vclocks: AtomicU32,
  vclock_mux: Mutex<()>,
  vclock_index: i32,                   // for vclock instances; -1 on physical
  worker: Option<KthreadWorker>,
  debugfs_root: Option<Dentry>,
}

struct ClockInfo {
  owner: ModuleRef,
  name: [u8; PTP_CLOCK_NAME_LEN],
  max_adj: i32,                        // ppb
  n_alarm: i32,
  n_ext_ts: i32,
  n_per_out: i32,
  n_pins: i32,
  pps: i32,
  pin_config: NonNull<[PinDesc]>,
  // callbacks: adjfine, adjphase, getmaxphase, adjtime, gettime64, gettimex64,
  // getcrosststamp, getcyclesx64, getcrosscycles, settime64, enable, verify,
  // do_aux_work
}

struct TimestampEventQueue {
  qlist: ListLink,
  buf: [PtpExttsEvent; PTP_MAX_TIMESTAMPS],
  head: u32,
  tail: u32,
  lock: SpinLock,
  mask: KBox<[u64]>,                   // PTP_MAX_CHANNELS bits
  debugfs_instance: Option<Dentry>,
  dfs_bitmap: DebugfsU32Array,
}
```

Registration (`ptp_clock_register`):
1. Allocate `ptp_clock`; assign `index = xa_alloc(&ptp_clocks_map, ptp, …)`.
2. Initialize `ptp->clock.ops = ptp_clock_ops`, `ptp->dev.class = &ptp_class`.
3. If `info->do_aux_work`, allocate kthread-worker.
4. `posix_clock_register(&ptp->clock, &ptp->dev)` — creates cdev + sysfs.
5. Return `ptp` to driver for use in `ptp_clock_event` calls.

Open (`ptp_open`):
1. Allocate `timestamp_event_queue`; allocate channel mask bitmap, default all-set.
2. Link into `ptp->tsevqs` under `tsevqs_lock`.
3. Create debugfs queue-instance.
4. Stash queue in `pccontext->private_clkdata`.

Ioctl dispatch (`ptp_ioctl`):
- `PTP_CLOCK_GETCAPS{,2}` → return `ptp_clock_caps` (max_adj, n_ext_ts, n_per_out, pps, n_pins, cross_timestamping, adjust_phase, max_phase_adj).
- `PTP_EXTTS_REQUEST{,2}` → validate `flags` against `PTP_EXTTS_VALID_FLAGS`; pin-mux setup via `ptp_set_pinfunc`; `info->enable(EXTTS, on)`.
- `PTP_PEROUT_REQUEST{,2}` → validate `flags` against `PTP_PEROUT_VALID_FLAGS`; pin-mux setup; `info->enable(PEROUT, on)`.
- `PTP_ENABLE_PPS{,2}` → `capable(CAP_SYS_TIME)`; `info->enable(PPS, on)`.
- `PTP_SYS_OFFSET{,2}` / `_PRECISE{,2}` / `_EXTENDED{,2,_CYCLES}` → cross-timestamp; `gettimex64`/`getcrosststamp` triplets.
- `PTP_PIN_GETFUNC{,2}` / `PTP_PIN_SETFUNC{,2}` → enumerate / configure pin-mux.
- `PTP_MASK_CLEAR_ALL` / `PTP_MASK_EN_SINGLE` → per-fd channel mask.

Read (`ptp_read`):
1. From per-fd queue, dequeue `n` `ptp_extts_event` under `queue->lock`.
2. `copy_to_user(buf, &queue->buf[head], n * sizeof(ev))`.

Event push (`ptp_clock_event`):
1. Under `tsevqs_lock`, for each per-fd queue: if `test_bit(ev->index, queue->mask)`, `enqueue_external_timestamp(queue, ev)`.
2. If `info->pps != 0` and ev is PPS, `pps_event` into pps-kernel.

Cross-timestamp `PTP_SYS_OFFSET_PRECISE`:
1. `info->getcrosststamp(info, &xtstamp)` → fills `system_realtime`, `device`, `sys_realtime`.
2. Copy back to user.

## Hardening

(Inherits from `drivers/ptp/00-overview.md`.)

ptp-core specific:

- **`ptp_clock_freerun` interlock** — once any vclock is registered, parent PHC rejects `settime`/`adjtime` from any context other than vclock infrastructure; defeats accidental time-warp from naive `linuxptp` against a vclock-active card.
- **per-channel mask validation** — `bitmap_set(mask, 0, PTP_MAX_CHANNELS)` default; per-fd modifications via `PTP_MASK_EN_SINGLE` are explicit; preserves least-surprise for legacy openers.
- **`PTP_STRICT_FLAGS` forced for `*_REQUEST2`** — V2 ioctls reject unknown flag bits at the kernel side, defeating silent flag-drift across kernel/userspace versions.
- **`adjfine` ppb clamp** — `scaled_ppm_to_ppb(tx->freq)` checked against `info->max_adj`; out-of-range returns `-ERANGE` rather than wrap.
- **`adjphase` `getmaxphase` clamp** — `tx->offset` checked against driver-advertised `max_phase_adj`; `-ERANGE` on overflow.
- **`timespec64_valid_settod` on settime** — non-monotonic settime values rejected; defeats negative-second / future-clamp injection.
- **`pccontext->private_clkdata` per-open allocation** — every queue is per-open; concurrent reads from multiple fds don't interfere.
- **defunct flag** — set on `ptp_clock_unregister` start; pending ioctls return `-ENODEV` after first re-entry; defeats UAF on hot-unplug.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `ptp_clock`, `timestamp_event_queue`, `ptp_clock_request`, `ptp_pin_desc[]`, and per-ioctl arg structs (`ptp_clock_caps`, `ptp_sys_offset*`, `ptp_extts_request*`, `ptp_perout_request*`).
- **PAX_KERNEXEC** — PTP core text W^X; `ptp_class`, `posix_clock_operations ptp_clock_ops`, and per-driver `ptp_clock_info` instances live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kstack across `ptp_ioctl`, `ptp_clock_event`, `ptp_clock_adjtime`, `ptp_set_pinfunc`, and per-PHC IRQ → event-push paths.
- **PAX_REFCOUNT** — saturating `refcount_t` on `ptp_clock`, on the underlying `device::kobj.kref`, on every vclock binding, and on `posix_clock` references; overflow trap defeats unregister-races on hot-unplugged NICs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `ptp_clock`, queue rings (carrying sub-µs timestamps that leak system-load profiling info), and per-pin descriptor arrays; vclock teardown scrubs prior tenant state.
- **PAX_UDEREF** — SMAP/PAN enforced on every ioctl entry; `copy_from_user`/`copy_to_user` with explicit struct sizes; per-`nospec` index sanitization on user-supplied `index`/`channel` fields (already present via `array_index_nospec` in upstream).
- **PAX_RAP / kCFI** — `posix_clock_operations`, `ptp_clock_info` (gettime64, gettimex64, settime64, adjfine, adjphase, adjtime, enable, verify, getcrosststamp, do_aux_work), and per-vclock fan-out callbacks marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-PHC frequency/phase-history disclosure behind CAP_SYSLOG; PHC adjustment trace + dynamic frequency are timing-side-channel signal.
- **GRKERNSEC_DMESG** — restrict PHC fault, vclock-bind, and freerun-transition banners to CAP_SYSLOG.
- **`/dev/ptp*` CAP_SYS_ADMIN policy** — `settime64` and `adjtime`/`adjfine`/`adjphase` (clock-adjust) require CAP_SYS_TIME at the POSIX-clock layer; this Tier-3 doc treats CAP_SYS_ADMIN as a policy upgrade (per Rookery hardening default) for pin-mux + PEROUT-program writes since they can disturb other tenants' phase tracking.
- **ioctl PAX_USERCOPY** — every `_IOW` / `_IOWR` payload structure has an explicit USERCOPY whitelist entry at the exact byte count documented in `ptp_clock.h`; defeats off-by-one struct-evolution underread/overread.
- **pin-mux CAP_SYS_RAWIO** — `PTP_PIN_SETFUNC{,2}` and `extts_enable`/`period`/`pps_enable` sysfs writes gated CAP_SYS_RAWIO (drives external SMA pins on time-cards / dedicated GPIO on NICs).
- **`PTP_STRICT_FLAGS` enforced for V2 ioctls** — unknown flag bits cause `-EINVAL`; defeats flag-drift attacks supplying future-version bits to current-version kernels.
- **vclock binding gated** — `max_vclocks` is admin-configured; `n_vclocks` writes require CAP_SYS_ADMIN; defeats unprivileged exhaustion of vclock pool.
- **freerun interlock** — parent settime/adjtime rejected with `-EBUSY` once vclocks bound; ensures vclock-based isolation cannot be subverted by a co-tenant adjusting the parent.
- **Event-queue overflow accounted** — `enqueue_external_timestamp` advances `head` on overflow (drop-oldest); never blocks an IRQ handler.

Rationale: PTP hardware clocks are the kernel's authoritative time-source for industrial / financial / 5G fronthaul / telemetry-attestation workloads; the same hardware also drives external pins that can wedge or noise-inject into co-located equipment. A spoofed `settime` against a parent PHC while vclocks were trusted would silently de-synchronize every dependent VLAN; a CAP-relaxed pin-mux could repurpose a board's SMA output (potentially flagged for emergency cellular sync) into a fuzz signal. RAP/kCFI on `ptp_clock_info`, CAP_SYS_TIME on clock adjust, CAP_SYS_RAWIO on pin-mux, strict-flags enforcement on V2 ioctls, freerun interlock, and refcount-overflow trapping turn the PTP framework from a thin POSIX-clock shim into a structurally bounded privileged surface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-driver PHC implementations (covered in their owning subsystem Tier-3s: `intel-ice-ptp.md`, `mlx5-ptp.md`, `ptp_ocp.md` future Tier-3s)
- Virtual PTP clocks (covered in `drivers/ptp/ptp_vclock.md` future Tier-3 if needed)
- `ptp_kvm` / `ptp_vmclock` for confidential VMs (future Tier-3 `ptp_kvm.md`)
- `linuxptp` userspace (`ptp4l`, `phc2sys`, `ts2phc`)
- IEEE 1588 protocol semantics (specification)
- DSA-switch PTP (covered in `net/dsa/` Tier-3)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `queue_no_oob` | OOB | `head` and `tail` always `< PTP_MAX_TIMESTAMPS`; wrap arithmetic preserved |
| `channel_index_bounded` | OOB | every `ev->index` < `info->n_ext_ts`; `nospec`-sanitized |
| `pin_mux_no_alias` | UNIQUENESS | at most one pin/channel pair owns a function at any time |
| `vclock_freerun_excludes_settime` | RACE | `n_vclocks > 0` ⇒ parent `settime` returns `-EBUSY` |

### Layer 2: TLA+

`models/ptp/vclock_freerun.tla`: proves the parent↔vclock state machine; once any vclock is bound, parent enters freerun; vclock unregister sequence returns parent to mutable state with no missed `freerun ↔ adjustable` transition.

### Layer 3: Verus invariants

- `ptp_clock_event` post: every per-fd queue whose mask matches `ev->index` receives exactly one event; queues not matching are untouched.
- `ptp_clock_adjtime` post: on success, exactly one of `adjtime`/`adjfine`/`adjphase` was invoked; freerun state unchanged.

### Layer 4: Functional

`linuxptp` regression suite (ptp4l + phc2sys + ts2phc) under KASAN + UBSAN; `testptp` exercising every ioctl variant against `ptp_mock`; concurrent vclock bind/unbind storm.
