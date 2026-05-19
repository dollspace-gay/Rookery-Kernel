# Tier-3: drivers/counter/{counter-core,counter-sysfs,counter-chrdev}.c — generic counter framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/counter/00-overview.md
upstream-paths:
  - drivers/counter/counter-core.c
  - drivers/counter/counter-sysfs.c
  - drivers/counter/counter-chrdev.c
  - drivers/counter/counter-chrdev.h
  - drivers/counter/counter-sysfs.h
  - include/uapi/linux/counter.h
  - include/linux/counter.h
-->

## Summary

The counter subsystem (`drivers/counter/`) provides a generic, hardware-agnostic framework for quadrature-encoders, pulse-counters, capture-compare units, and other counting hardware (industrial encoders, motor-control feedback, frequency counters, time-base captures). A counter driver registers `struct counter_device` describing N `counter_count` (counts) each fed by M `counter_signal` (signals) wired through K `counter_synapse` (signal-to-count routings); the core exports per-device `/sys/bus/counter/devices/counterN/` attribute hierarchy and `/dev/counterN` char-device for event watch + read.

This Tier-3 covers `drivers/counter/counter-core.c` (~283 lines: device alloc/add/register lifecycle, IDA-managed device-id), `counter-sysfs.c` (~1174 lines: full sysfs attribute matrix per count/signal/synapse/extension), and `counter-chrdev.c` (~675 lines: `/dev/counterN` poll/read/ioctl event-watch interface). Per-controller drivers (`104-quad-8`, `i8254`, `intel-qep`, `interrupt-cnt`, `ftm-quaddec`, `microchip-tcb-capture`, `rz-mtu3-cnt`, `stm32-timer-cnt`, `stm32-lptimer-cnt`, `ti-ecap-capture`, `ti-eqep`) consume this contract.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct counter_device` (`include/linux/counter.h`) | per-device descriptor (parent, ops, counts[], signals[], num_*, ext[]) | `drivers::counter::Device` |
| `struct counter_ops` | per-driver vtable (`signal_read`, `count_read`, `count_write`, `function_read`/`_write`, `action_read`/`_write`, `events_configure`, `watch_validate`) | `drivers::counter::Ops` |
| `struct counter_count` | per-count descriptor (id, name, functions_list, synapses[], ext[]) | `drivers::counter::Count` |
| `struct counter_signal` | per-signal descriptor (id, name, ext[]) | `drivers::counter::Signal` |
| `struct counter_synapse` | (signal_ref, count) routing with allowed `counter_synapse_action[]` | `drivers::counter::Synapse` |
| `counter_alloc(sizeof_priv)` (`counter-core.c`) | alloc + IDA assign | `Device::alloc` |
| `counter_add(counter)` | register sysfs + chrdev | `Device::add` |
| `counter_unregister(counter)` | unregister | `Device::unregister` |
| `counter_put(counter)` | refcount drop | `Device::put` |
| `devm_counter_alloc(dev, sz)` / `devm_counter_add(dev, c)` | devm wrappers | `Device::devm_alloc` / `_add` |
| `counter_priv(counter)` | driver private pointer | `Device::priv` |
| `counter_push_event(counter, event, channel)` (`counter-chrdev.c`) | enqueue event for `/dev/counterN` listeners | `Device::push_event` |
| `counter_chrdev_add(counter)` / `_remove(counter)` | per-device chrdev plumbing | `Device::chrdev_add` / `_remove` |
| `counter_sysfs_add(counter)` (`counter-sysfs.c`) | per-device sysfs tree | `Device::sysfs_add` |
| `struct counter_watch` (`uapi/linux/counter.h`) | userspace watch descriptor | `uapi::CounterWatch` |
| `struct counter_event` | per-event push descriptor delivered via read() | `uapi::CounterEvent` |
| `COUNTER_ADD_WATCH_IOCTL` / `_ENABLE_EVENTS_IOCTL` / `_DISABLE_EVENTS_IOCTL` | ioctls | `uapi::CounterIoctl` |

## Compatibility contract

REQ-1: Per-device id assigned from `counter_ida` (IDA-managed); `/dev/counterN` minor is the same id. `COUNTER_DEV_MAX = 256` cap.

REQ-2: Per-device `counter_ops` vtable must provide at least one of `count_read`/`_write`; per-count `function_read`/`_write` if multiple `counter_function` (e.g. `INCREASE`, `DECREASE`, `PULSE_DIRECTION`, `QUADRATURE_X1_*`, `_X2_*`, `_X4_*`) are supported; per-synapse `action_read`/`_write` if `counter_synapse_action` (e.g. `NONE`, `RISING_EDGE`, `FALLING_EDGE`, `BOTH_EDGES`) is configurable.

REQ-3: sysfs surface `/sys/bus/counter/devices/counterN/` exposes `name`, `num_counts`, `num_signals` (R/O); per-count `count<i>/{name,count,function,functions_available,ceiling,floor,preset,enable,…}`; per-synapse `count<i>/signal<j>_action[_available]`; per-signal `signal<j>/{name,value,polarity[_available]}`; per-`counter_comp` extension `<name>[_available]` pairs (see `enum counter_component_type`).

REQ-4: `/dev/counterN` char-device — `read(2)` blocking read of `struct counter_event` (16-byte records: `timestamp` + `value` + `watch.id`); `poll(2)` POLLIN when queue non-empty; `ioctl(2)` `COUNTER_ADD_WATCH_IOCTL(struct counter_watch)` to register a watch (per-watch carries `event_type`, `channel_type`, `parent_type`, `parent`, `id`), `COUNTER_ENABLE_EVENTS_IOCTL` to commit pending watch-list to active, `COUNTER_DISABLE_EVENTS_IOCTL` to clear.

REQ-5: Event types (`enum counter_event_type`): `COUNTER_EVENT_OVERFLOW`, `_UNDERFLOW`, `_OVERFLOW_UNDERFLOW`, `_THRESHOLD`, `_INDEX`, `_CHANGE_OF_STATE`, `_CAPTURE`.

REQ-6: Watch validation — per-driver `ops.watch_validate(counter, &watch)` invoked at `COUNTER_ADD_WATCH_IOCTL`; refuses unsupported `(event_type, channel)` combinations. Default = accept-all.

REQ-7: Per-device event queue is bounded by a `kfifo` sized at chrdev-add; overflow drops the newest event + sets `COUNTER_EVENT_OVERFLOW` flag on the next delivered event.

REQ-8: `counter_push_event(counter, event, channel)` is callable from IRQ context — drivers call this in their capture/threshold ISR; the core fans out to per-watch matches and wakes pollers.

REQ-9: Per-device extension components (`counter_comp`) declared in `count[].ext[]`, `signal[].ext[]`, or `counter.ext[]`; per-component type ∈ `enum counter_component_type` (`SIGNAL`, `COUNT`, `FUNCTION`, `SYNAPSE_ACTION`, `EXTENSION`).

REQ-10: Counter values are `u64` (driver-decided unit — typically a hardware-count-register reading); sysfs renders as decimal; uapi `counter_event.value` is `u64`.

REQ-11: Symbol namespace — every export is `EXPORT_SYMBOL_NS_GPL(name, "COUNTER")`; consumer must `MODULE_IMPORT_NS("COUNTER")`.

## Acceptance Criteria

- [ ] AC-1: A test driver registering one count + one signal + one synapse appears at `/sys/bus/counter/devices/counter0/` with the expected attribute tree.
- [ ] AC-2: `/dev/counter0` major/minor matches the chrdev allocation; `stat /dev/counter0` returns `c` mode.
- [ ] AC-3: Userspace `read()` of `/dev/counter0` without prior `COUNTER_ADD_WATCH` blocks on empty queue (POLLIN cleared).
- [ ] AC-4: Userspace registers `counter_watch{event=COUNTER_EVENT_OVERFLOW, channel=0}` via `COUNTER_ADD_WATCH_IOCTL`, then `COUNTER_ENABLE_EVENTS_IOCTL`; on driver-triggered overflow, `read()` returns a `counter_event` with matching `id` + `timestamp` + `value`.
- [ ] AC-5: `cat /sys/bus/counter/devices/counter0/count0/count` returns current count value; `echo 0 > .../count` works if driver supports `count_write`.
- [ ] AC-6: Watch overflow (more events than queue depth) sets `COUNTER_EVENT_OVERFLOW` flag on next delivered event.
- [ ] AC-7: Unsupported watch (event_type not in driver mask) → `COUNTER_ADD_WATCH_IOCTL` returns -EINVAL via `ops.watch_validate`.
- [ ] AC-8: Concurrent `read()` from two opens delivers each event to exactly one reader (or driver opts into broadcast).
- [ ] AC-9: `IDA_MAX_COUNTER_DEVICES = 256` (COUNTER_DEV_MAX) — 257th `counter_add` returns -ENOMEM cleanly.

## Architecture

`Subsystem` lives in `drivers::counter::Subsystem`:

```
struct Subsystem {
  bus: KBox<BusType>,                  // "counter"
  ida: Ida,
  major: u32,                          // chrdev major for /dev/counterN
}

struct Device {
  refcount: Refcount,
  id: u32,
  parent: NonNull<KDevice>,
  ops: &'static Ops,
  signals: &'static [Signal],
  num_signals: usize,
  counts: &'static [Count],
  num_counts: usize,
  ext: &'static [Component],
  priv_ptr: NonNull<u8>,
  chrdev: KBox<Chrdev>,
  events: KFifo<Event>,                // per-device event queue
  events_wait: WaitQueue,
  events_lock: SpinLock<()>,           // protects events kfifo
  events_in_use: AtomicBool,           // set by ENABLE_EVENTS_IOCTL
  next_events_lock: Mutex<()>,
  next_events_list: List<WatchEntry>,  // pending ADD_WATCH list
  events_list: List<WatchEntry>,       // active (post-ENABLE) list
}

struct WatchEntry {
  watch: CounterWatch,                 // uapi
  comp: &'static Component,            // resolved
  next: ListNode,
}
```

Lifecycle (`drivers/counter/counter-core.c`):

1. `counter_alloc(sizeof_priv)`:
   - `ida_alloc_max(&counter_ida, COUNTER_DEV_MAX-1, GFP_KERNEL)` → id.
   - `kzalloc(sizeof(struct counter_device) + sizeof_priv)`.
   - Initialize embedded `struct device` with class = counter_class, parent = NULL until `counter_add`.
   - Init `refcount`, `events_wait`, `next_events_list`, `events_list`.
   - Return.

2. `counter_add(counter)`:
   - Validate `ops` shape per signal/count/synapse declarations.
   - Set `dev->parent = counter->parent`.
   - `dev_set_name(dev, "counter%d", id)`.
   - `device_add(dev)`.
   - `counter_sysfs_add(counter)` — build attribute groups for every count/signal/synapse/extension.
   - `counter_chrdev_add(counter)` — cdev_init + cdev_add(major, id, 1).

3. `counter_unregister(counter)` / `counter_put(counter)`:
   - `counter_chrdev_remove` → cdev_del.
   - `counter_sysfs_remove` → device_remove_groups.
   - `device_del`.
   - `ida_free(&counter_ida, id)`.
   - kfree on refcount==0.

sysfs surface (`counter-sysfs.c`):
- For each `counter->counts[i]`: attribute group with `count`, `function`, `functions_available`, `ceiling`, `floor`, `preset`, `enable`, … as supported.
- For each synapse of count: `signal<j>_action`, `signal<j>_action_available`.
- For each `counter->signals[j]`: `value`, `polarity`, …
- For each extension `counter->ext[k]`: `<name>` + `<name>_available`.

chrdev path (`counter-chrdev.c`):

- `counter_chrdev_open`: `filp->private_data = container_of(inode->i_cdev, ..., chrdev)`; `nonseekable_open`.
- `counter_chrdev_read`: loop `kfifo_out(events)` under `events_lock` (irqsave); on empty + !O_NONBLOCK `wait_event_interruptible(events_wait, !kfifo_is_empty)`.
- `counter_chrdev_poll`: `poll_wait(filp, &events_wait, wait)`; return POLLIN | POLLRDNORM if non-empty.
- `counter_chrdev_ioctl`:
  - `COUNTER_ADD_WATCH_IOCTL`: `copy_from_user(&watch, arg, sizeof(watch))`; resolve `watch.component` → counts/signals/ext pointer; `ops.watch_validate(counter, &watch)` may refuse with -EINVAL; allocate `WatchEntry`, append to `next_events_list` under `next_events_lock`.
  - `COUNTER_ENABLE_EVENTS_IOCTL`: swap `events_list` ← `next_events_list`, drain old, `events_in_use = true`, `ops.events_configure(counter)` arms HW IRQs.
  - `COUNTER_DISABLE_EVENTS_IOCTL`: clear `events_in_use`, drain, call `events_configure` with empty list.

`counter_push_event(counter, event, channel)` (IRQ-safe):
1. `spin_lock_irqsave(&events_lock)`.
2. Walk `events_list`; for each `WatchEntry` matching `(event, channel)`:
   - Build `struct counter_event{ timestamp = ktime_get_ns(), value = (count read via ops.count_read or 0), watch = entry.watch }`.
   - `kfifo_in(events, &ev, 1)`; if full, set `dropped` flag.
3. `spin_unlock_irqrestore`.
4. `wake_up_interruptible(&events_wait)`.

## Hardening

- **Per-device IDA cap** — `COUNTER_DEV_MAX = 256` enforced; refuse further `counter_add` past cap.
- **ioctl input bound** — `COUNTER_ADD_WATCH_IOCTL` copy `sizeof(struct counter_watch)` only; no variable-length user data.
- **Watch validation** — `ops.watch_validate` allows driver to refuse unsupported combinations.
- **Per-event kfifo bounded** — events_list overflow drops + sets dropped flag; no unbounded kernel allocation.
- **chrdev minor strict** — `cdev_add(major, id, 1)` per device; collisions impossible due to IDA.
- **events_lock IRQ-safe** — `spin_lock_irqsave` on push_event; safe from any IRQ context.
- **Driver private namespace** — `EXPORT_SYMBOL_NS_GPL(..., "COUNTER")`; non-GPL or non-importing modules refused.
- **sysfs attribute matrix RO/RW per-driver** — `ops` vtable absence makes attribute RO.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `counter_device`, `counter_event` kfifo backing, `WatchEntry`, and per-component caches; `counter_event` copy_to_user bounded by 16-byte record size.
- **PAX_KERNEXEC** — counter core in W^X kernel text; `counter_ops` tables, sysfs attribute groups, and chrdev `file_operations` live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `counter_push_event`, chrdev read/ioctl, and per-driver capture-ISR entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `counter_device` and per-watch-entry refs; overflow trap defeats add/unregister race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for events kfifo backing, watch-entry slabs, and driver-private regions so timing/index traces cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every chrdev read/poll/ioctl + every sysfs store callback; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `counter_ops` vtable (`count_read`/`_write`/`function_*`/`action_*`/`events_configure`/`watch_validate`) and chrdev `file_operations` marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-`counter_device` priv-ptr disclosure behind CAP_SYSLOG; suppress `%p` in events tracepoints.
- **GRKERNSEC_DMESG** — restrict per-driver overflow/underflow/threshold banners to CAP_SYSLOG so attackers cannot probe industrial-control-loop timing via dmesg.
- **/dev/counterN CAP_SYS_RAWIO** — open(2) of `/dev/counterN` requires CAP_SYS_RAWIO (or equivalent per-device ACL via udev rule); industrial-IO surface refused to unprivileged userspace.
- **counter_event PAX_USERCOPY whitelist** — events kfifo slab marked usercopy-whitelist only over the 16-byte `struct counter_event` window; refuse out-of-window copies.
- **Watch ioctl size cap** — `COUNTER_ADD_WATCH_IOCTL` copy strictly `sizeof(struct counter_watch)`; arg-length mismatch returns -EINVAL without allocation.
- **sysfs RO for non-root** — every R/W counter attribute (`count_write`, `preset`, `enable`, `function`, `action`, `polarity`) gated by file-mode 0600 + owner root, or per-device udev policy granting a specific industrial-control group; unprivileged write returns -EACCES.
- **Watch list bounded** — `next_events_list` + `events_list` per-device entry count capped; refuse `COUNTER_ADD_WATCH` past cap with -ENOSPC.
- **events_configure idempotent** — re-enable from same state must not double-arm HW IRQs; defense against `ENABLE_EVENTS_IOCTL` loop DoS.

Rationale: counter devices typically feed motor-control, factory-automation, and time-base loops where a malicious or buggy unprivileged process tampering with `preset`, `function`, or `action` attributes can drive a physical actuator into a fault state. /dev/counterN open without CAP_SYS_RAWIO, sysfs write without root, and unbounded watch-list growth are all real attack surfaces in industrial deployments. RAP/kCFI on `counter_ops` + chrdev `file_operations`, CAP_SYS_RAWIO on `/dev/counterN`, sysfs RO for non-root, watch-list bound, PAX_USERCOPY whitelist on the 16-byte event window, and refcount-overflow trapping turn the counter framework from "industrial-IO with default-Linux permissions" into a structurally policed boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `counter_ida_no_oob` | OOB | per-device id < COUNTER_DEV_MAX. |
| `event_kfifo_no_oob` | OOB | kfifo head/tail bounded; full → drop+flag, never wraps over live data. |
| `watch_resolve_no_oob` | OOB | watch.component → counts[] / signals[] / ext[] indices in range. |
| `chrdev_read_no_uaf` | UAF | `filp->private_data` outlives every concurrent read; unregister waits for opens. |

### Layer 2: TLA+

`models/counter/event_pipeline.tla` (this doc declares it): proves the kfifo-based event delivery is loss-aware (dropped flag) but never delivers stale-after-disable events; ENABLE_EVENTS_IOCTL ↔ DISABLE_EVENTS_IOCTL state-machine deadlock-free.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Device::add` post: device registered with bus="counter" ∧ chrdev_minor == id | `counter_add` |
| `Device::push_event` post: event delivered ∨ dropped flag set ∨ no listener match | `counter_push_event` |
| chrdev entry pre: caller has open(2) handle ∧ owning user has CAP_SYS_RAWIO (or device ACL) | every `counter_chrdev_*` |

### Layer 4: Verus/Creusot functional

Userspace open `/dev/counter0` → `COUNTER_ADD_WATCH_IOCTL(event=OVERFLOW, channel=0)` → `COUNTER_ENABLE_EVENTS_IOCTL` → driver capture ISR → `counter_push_event(counter, OVERFLOW, 0)` → events kfifo enqueue → wake → userspace `read()` returns 16-byte event. Encoded as Verus invariant chained with per-driver Tier-3 (e.g. `ti-eqep.md`).

## Out of Scope

- Per-driver internals (`104-quad-8`, `intel-qep`, `stm32-timer-cnt`, …) — future per-driver Tier-3s
- Industrial-IO `iio` subsystem (covered by `drivers/iio/` Tier-3s)
- Counter-specific ABI documentation (mirrored in `Documentation/ABI/testing/sysfs-bus-counter`)
- 32-bit-only paths
- Implementation code
