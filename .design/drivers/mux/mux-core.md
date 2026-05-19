# Tier-3: drivers/mux/core.c — generic multiplexer framework (states, exclusive vs shared selection)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/mux/core.c
  - drivers/mux/gpio.c
  - drivers/mux/mmio.c
  - drivers/mux/adg792a.c
  - drivers/mux/adgs1408.c
  - include/linux/mux/consumer.h
  - include/linux/mux/driver.h
-->

## Summary

The `mux` subsystem abstracts physical signal multiplexers — GPIO-controlled analog muxes (ADG792A, ADGS1408), MMIO-register pinmux helpers, regulator-controlled muxes — so consumer drivers (I2C bus multiplexer, ADC channel selector, audio jack matrix) can `mux_control_select(N)` an integer state without owning the per-chip register layout.

Each physical chip registers a `mux_chip` containing one or more `mux_control` instances (a 4:1 mux is one control with 4 states; a 4-channel mux array is four controls each with 4 states). Consumers obtain controls via DT phandles (`mux-controls`/`mux-control-names`) and either own the control exclusively (`mux_control_select`) or share it cooperatively (`mux_state_select` introduced for trylock-style usage). Per-control "idle state" auto-restores after release.

This Tier-3 covers `drivers/mux/core.c` (~899 lines: the framework + consumer API + sysfs class) plus the consumer + driver headers.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mux_chip` | per-physical-chip control block (1..N controls) | `drivers::mux::Chip` |
| `struct mux_control` | per-channel state machine + lock | `drivers::mux::Control` |
| `struct mux_state` | per-consumer `(control, state)` reservation handle | `drivers::mux::State` |
| `struct mux_control_ops` | `set(mux, state)` vtable | `drivers::mux::ControlOps` |
| `mux_chip_alloc(dev, controls_in_chip, sizeof_priv)` | per-chip alloc with embedded controls + priv | `Chip::alloc` |
| `mux_chip_register(mux_chip)` / `mux_chip_unregister(mux_chip)` | register + sysfs class | `Chip::register` / `_unregister` |
| `mux_chip_free(mux_chip)` | drop unregistered chip (mid-probe error) | `Chip::free` |
| `devm_mux_chip_alloc(dev, ...)` / `devm_mux_chip_register(dev, mux_chip)` | devres variants | `Chip::devm_*` |
| `mux_control_states(mux)` | query control's state count | `Control::states` |
| `mux_control_select_delay(mux, state, delay_us)` / `mux_state_select_delay(mstate, delay_us)` | blocking exclusive select with post-set settle | `Control::select_delay` |
| `mux_control_try_select_delay(...)` / `mux_state_try_select_delay(...)` | non-blocking variants | `Control::try_select_delay` |
| `mux_control_deselect(mux)` / `mux_state_deselect(mstate)` | release; optionally restore idle state | `Control::deselect` |
| `mux_control_get(dev, name)` / `mux_control_get_optional(dev, name)` / `devm_mux_control_get(dev, name)` | DT phandle resolution | `Control::get` |
| `mux_state_get(dev, name)` / `mux_state_get_optional(dev, name)` / `devm_mux_state_get(dev, name)` | per-consumer state-handle resolution (DT `mux-states` + `mux-state-names`) | `State::get` |
| `mux_control_put(mux)` / `mux_state_put(mstate)` | drop reference | `Control::put` / `State::put` |

## Compatibility contract

REQ-1: Chip lifecycle: `mux_chip_alloc(dev, controls, priv_bytes)` allocates a per-chip block with N embedded `mux_control` + per-driver private area; `mux_chip_register` adds the chip to the `mux_class` sysfs class (`/sys/class/mux/muxchip%d/`).

REQ-2: Per-control state count + idle-state set at probe by the driver via `mux->states` and `mux->idle_state` (range `[0, states-1]` or `MUX_IDLE_AS_IS` / `MUX_IDLE_DISCONNECT`).

REQ-3: Per-control HW set: driver `ops->set(mux, state)` invoked under `mux->lock`; per-call return non-zero on HW failure (e.g. I2C error) — core surfaces error to caller.

REQ-4: Per-control reservation: `mux_control` carries `mutex` (exclusive) + `state` cache; `mux_control_select` is blocking lock + set; `_deselect` releases lock + optionally restores idle.

REQ-5: `mux_state_get` returns a per-consumer `(control, state)` pair parsed from DT `mux-states = <&mux 0 &mux 2>` + `mux-state-names = "rx" "tx"`; consumer never owns the integer-state separately from the handle.

REQ-6: DT phandle parsing: `#mux-control-cells` (typically 1 for chips with multiple controls, 0 for single-control chips) determines argument arity; out-of-range control index returns `-EINVAL`.

REQ-7: Idle behavior: on `_deselect`, if `idle_state != MUX_IDLE_AS_IS`, core re-invokes `ops->set(mux, idle_state)`; for `MUX_IDLE_DISCONNECT` (typical for analog muxes), core selects `states` (out-of-range value signifying "all channels off").

REQ-8: try-select semantics: `mux_control_try_select_delay` uses `mutex_trylock`; if locked, returns `-EBUSY` immediately.

REQ-9: Per-control `select_delay_ns` field: after `ops->set` returns, the core sleeps for the configured per-control delay (typically the mux's propagation time) before returning to caller.

REQ-10: Reference counting: per-control refcount via `kref` on the embedding chip; `mux_control_put` releases the per-consumer reference; chip teardown blocked while any control holds active consumers.

REQ-11: Sysfs class: read-only `state` attribute exposes current cached state; `mux-class/muxchip%d/state` requires class membership.

REQ-12: Module lifetime: per-chip `THIS_MODULE` recorded so `mux_control_get` raises module refcount on the providing driver, preventing rmmod while a consumer holds a control.

## Acceptance Criteria

- [ ] AC-1: I2C bus multiplexer (`drivers/i2c/muxes/i2c-mux-gpmux.c`) consuming `mux_control_select` on a GPIO-mux-backed control routes I2C traffic to expected child bus on selection.
- [ ] AC-2: ADC channel selection on `iio` ADC + analog mux yields per-channel readings matching expected mux state.
- [ ] AC-3: Two consumers contesting the same exclusive `mux_control`: first selects, second blocks until first's `_deselect`; trylock variant returns `-EBUSY`.
- [ ] AC-4: Idle-state restoration: `mux_control_deselect` returns control to `idle_state` observable on sysfs `state` attribute.
- [ ] AC-5: Disconnect idle on ADG792A: after deselect, all channels open as expected (verified by per-channel continuity probe).
- [ ] AC-6: DT phandle resolution: `mux-controls` + `mux-control-names` parse rejects out-of-range index with `-EINVAL`.
- [ ] AC-7: rmmod of providing driver while consumer holds control fails with `-EBUSY`; release of consumer then allows rmmod.
- [ ] AC-8: KUnit (none for mux currently, but unit tests covering state transitions pass).

## Architecture

`Chip` + `Control` core layout:

```
struct Chip {
  dev: Device,
  id: u32,                       // class-allocated muxchipN id
  controls: u32,                 // number of controls in this chip
  mux: Vec<Control>,             // embedded array
  priv_size: usize,
}

struct Control {
  chip: Arc<Chip>,
  ops: &'static ControlOps,
  states: u32,
  cached_state: i32,             // -1 = unknown
  idle_state: IdleState,
  select_delay_ns: u32,
  lock: Mutex<()>,
  refcnt: Kref,
}

enum IdleState {
  AsIs,                          // MUX_IDLE_AS_IS
  Disconnect,                    // MUX_IDLE_DISCONNECT
  State(u32),                    // explicit
}

struct State {
  mux: Arc<Control>,
  state: u32,
}

trait ControlOps {
  fn set(mux: &Control, state: u32) -> Result<()>;
}
```

Chip alloc `mux_chip_alloc(dev, controls, priv_bytes)`:
1. Allocate `mux_chip` + `controls * sizeof(mux_control)` + `priv_bytes` inline (single allocation, contiguous layout).
2. Initialize `device` template with `mux_class`, `dev->parent = dev`.
3. Initialize each `mux_control[i]` with `chip = mux_chip`, `idle_state = MUX_IDLE_AS_IS`, `cached_state = -1`.
4. Return `mux_chip` for the driver to fill in `ops` + `states` per control.

Register `mux_chip_register`:
1. `ida_simple_get(&mux_ida, ...)` for muxchip id.
2. `dev_set_name(&chip->dev, "muxchip%u", id)`.
3. `device_register(&chip->dev)` against `mux_class`.
4. For each control with `idle_state != AS_IS`: invoke `ops->set(mux, idle_state.value)` (with `disconnect` mapped to `states`).

Consumer lookup `mux_control_get(dev, name)`:
1. Parse `mux-controls` + `mux-control-names` from `dev->of_node`.
2. Find index matching `name` (or 0 if name is NULL).
3. `of_parse_phandle_with_args(np, "mux-controls", "#mux-control-cells", idx, &args)`.
4. `mux_chip` looked up via `class_find_device` matching `of_node = args.np`.
5. Validate `args.args[0] < chip->controls`; return `&chip->mux[args.args[0]]`.
6. `try_module_get(chip->dev.parent->driver->owner)` to pin providing module.

Exclusive select `mux_control_select_delay(mux, state, delay_us)`:
1. Validate `state < mux->states`.
2. `mutex_lock(&mux->lock)`.
3. If `mux->cached_state != state`: `ret = mux->ops->set(mux, state)`; on error, unlock + return.
4. `mux->cached_state = state`.
5. If `delay_us > 0`: `fsleep(delay_us)` (or `udelay` < 10 us).
6. Return success — caller holds the lock until `_deselect`.

Try-select: as above but `mutex_trylock`; if `-EBUSY`, return immediately without state change.

Deselect `mux_control_deselect`:
1. If `idle_state != AS_IS`: `mux->ops->set(mux, idle_value)`; `cached_state = idle_value`.
2. `mutex_unlock(&mux->lock)`.

State-handle API `mux_state_select_delay(mstate, delay)`: thin wrapper invoking `mux_control_select_delay(mstate->mux, mstate->state, delay)`; the consumer never sees raw integer state.

Put `mux_control_put`: `kref_put` on chip; on last reference, `module_put(provider)`. `mux_chip_unregister` blocks if any control's `kref` > 0 transitively via chip refcount.

## Hardening

- **Per-control `mutex` + `cached_state`** — guarantees `ops->set` callbacks serialize per control; no two consumers race the underlying GPIO/MMIO.
- **State arg bounded** — `state < mux->states` checked before every `ops->set` invocation.
- **Module pin via `try_module_get`** — defense against rmmod of the providing driver while a consumer holds a control.
- **DT parse strict** — `#mux-control-cells` validated; OOB arg indexes rejected.
- **Idle-state restoration on deselect** — defense against leaving HW in a non-idle state that surprises the next consumer.
- **`mux_chip_alloc` single contiguous allocation** — defense against per-control alloc-failure mid-init causing inconsistent chip state.
- **Class-bound device** — sysfs entries only created on successful register; mid-probe error path uses `mux_chip_free` (no class device).
- **trylock variant separate** — non-blocking callers explicitly opt in; default exclusive `_select` always blocks.
- **Per-chip refcount drops module pin last** — providing driver cannot be unloaded while controls exist.
- **Sysfs `state` attribute read-only** — userspace cannot bypass kernel selection logic by writing state.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `mux_chip`, `mux_control`, `mux_state` and per-driver private blobs; reject userspace copy of state outside fixed-size sysfs attribute helpers.
- **PAX_KERNEXEC** — mux core in W^X kernel text; per-driver `mux_control_ops` placed in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `mux_control_get`, `mux_control_select`, `mux_control_deselect`, and `mux_chip_register` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on chip refcount, per-control `kref`, and per-consumer `mux_state`; defense against teardown-vs-select races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `mux_chip` (including embedded controls + priv area), `mux_state` reservations, and DT-parsed phandle-args buffers so prior per-control state cannot bleed to next allocation.
- **PAX_UDEREF** — SMAP/PAN enforced on sysfs entry; reject user-pointer deref outside canonical `device_attribute` show helpers.
- **PAX_RAP / kCFI** — `mux_control_ops->set` and any per-driver helper vtable marked `__ro_after_init` and dispatched via kCFI-typed indirect calls; the framework's per-class `dev->release` likewise.
- **GRKERNSEC_HIDESYM** — gate disclosure of per-chip `priv` pointer + per-control internal addresses (in `/proc/iomem`, debug prints) behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict mux register/deregister + per-set HW-failure banners to CAP_SYSLOG so attackers cannot probe per-control transitions via dmesg.
- **Chip-state PAX_REFCOUNT** — `mux_chip` refcount via `kref` saturating; mux_chip_unregister refused if non-zero refcount.
- **Lookup-args bounded** — `of_parse_phandle_with_args` results validated: `args.args[0] < chip->controls` enforced; OOB index trapped not silently truncated.
- **`mux_control_ops` kCFI** — every `ops->set` call goes through kCFI-typed indirect dispatch; vtable swap defeated.
- **gpio-of-bind validated** — `drivers/mux/gpio.c` GPIO-mux driver validates per-GPIO descriptor count matches state-count (log2 ceil) at probe; refuse mismatched DT.
- **Module pin enforced** — provider driver cannot be rmmod'd while any consumer holds a control; the `try_module_get` path is non-bypassable.
- **Sysfs attributes restricted** — `state` is RO; writable attrs (none today) would need CAP_SYS_RAWIO if added.

Rationale: muxes route physical signals — getting the state wrong on an analog-mux-fronted ADC can short-circuit channels; getting it wrong on an I2C bus mux can leak commands to the wrong slave. PAX_REFCOUNT on chip + control + state handles, kCFI on `ops->set`, bounded state args, and DT-parse strictness keep a misconfigured DT or a buggy provider driver from corrupting the framework's view of HW state. Module-pin enforcement closes the obvious "unload provider mid-select" UAF.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-chip drivers (`adg792a.c`, `adgs1408.c`, `gpio.c`, `mmio.c`) — each gets a future Tier-3 if depth warranted; baseline grsec inherits from this doc.
- I2C bus multiplexer integration (`drivers/i2c/muxes/`) covered under that subsystem.
- 32-bit-only paths.
- Implementation code.
