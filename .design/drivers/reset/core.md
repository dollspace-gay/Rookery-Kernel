# Tier-3: drivers/reset/core.c — reset controller framework (DT/fwnode binding, assert/deassert refcounts, shared vs exclusive, GPIO-reset adapter)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/reset/00-overview.md
upstream-paths:
  - drivers/reset/core.c
  - include/linux/reset.h
  - include/linux/reset-controller.h
  - include/linux/reset/reset-simple.h
  - Documentation/devicetree/bindings/reset/
-->

## Summary

The reset controller framework — a thin DT/fwnode-routed abstraction over per-SoC "drive a wire active to put device X into reset" hardware. SoC vendors expose dozens to hundreds of "reset lines" (one per peripheral block) typically aggregated under a single MMIO register file, where each bit asserts/deasserts the reset of one block. Consumer drivers (USB, I2C, MMC, networking, etc.) call `reset_control_assert(rstc)` / `_deassert(rstc)` / `_reset(rstc)` to pulse the line during init, suspend, or error-recovery; the framework dispatches to the registered controller's `reset_control_ops` callbacks.

The core also implements **shared** vs **exclusive** semantics (multiple consumers vs single owner of a line), **acquire/release** for transferable ownership (e.g. PHY hand-off between USB controllers), **bulk** API for arrays of related resets, an **array** wrapper for atomic multi-line operations, and an **auxiliary GPIO-reset adapter** that synthesizes a reset controller from a `reset-gpios` DT property when the platform has no MMIO reset block but a GPIO pin gates the device.

This Tier-3 covers `drivers/reset/core.c` (~1600 lines: registration, lookup/xlate, assert/deassert ref-counting, array + bulk paths, GPIO-reset aux device, SRCU-protected rcdev pointer for hot-unplug).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct reset_controller_dev` | per-controller registration record: ops, owner, fwnode/of_node, nr_resets, xlate | `drivers::reset::ControllerDev` |
| `struct reset_control` | per-consumer handle: rcdev pointer (SRCU-protected), id, kref, shared/acquired flags, deassert_count, triggered_count | `drivers::reset::Control` |
| `struct reset_control_array` | array of per-line controls wrapped as a composite handle | `drivers::reset::ControlArray` |
| `struct reset_control_ops` | per-driver vtable: `.reset`/`.assert`/`.deassert`/`.status` | `ResetControlOps` trait |
| `reset_controller_register(rcdev)` / `reset_controller_unregister(rcdev)` | per-driver registration | `ControllerDev::register` / `_unregister` |
| `devm_reset_controller_register(dev, rcdev)` | devres-managed variant | `ControllerDev::devm_register` |
| `__reset_control_get(dev, id, index, flags)` | per-consumer lookup by DT name or index | `Control::get` |
| `__fwnode_reset_control_get(fwnode, ...)` | fwnode-rooted lookup | `Control::get_fwnode` |
| `__devm_reset_control_get(dev, id, index, flags)` | devres-managed get | `Control::devm_get` |
| `reset_control_put(rstc)` | per-consumer drop; kref_put → __reset_control_release | `Control::put` |
| `reset_control_reset(rstc)` | pulse: assert then deassert (or controller's `.reset` if defined); shared lines pulse once | `Control::reset` |
| `reset_control_rearm(rstc)` | re-allow a shared reset to fire again on the next consumer's reset() | `Control::rearm` |
| `reset_control_assert(rstc)` / `reset_control_deassert(rstc)` | per-line assert/deassert with deassert_count balancing | `Control::assert` / `_deassert` |
| `reset_control_status(rstc)` | read-back line state | `Control::status` |
| `reset_control_acquire(rstc)` / `reset_control_release(rstc)` | transferable-ownership handshake for exclusive lines | `Control::acquire` / `_release` |
| `reset_control_bulk_get_*` / `_bulk_assert/_deassert/_reset/_acquire/_release` | array-API for related groups of resets | `Control::bulk_*` |
| `of_reset_control_get_count(np)` / `reset_control_get_count(dev)` | how many resets a consumer DT node references | `Control::get_count` |
| `fwnode_reset_simple_xlate` | default 1-cell `<id>` translation | `ControllerDev::default_xlate` |
| `reset_create_gpio_aux_device(rgpio, parent)` / `reset_add_reset_gpio_device(fwnode, args)` | synthesize a reset controller from `reset-gpios` DT property via auxiliary bus | `ControllerDev::create_gpio_aux` |

## Compatibility contract

REQ-1: Each driver populates a `struct reset_controller_dev` with `.ops`, `.owner = THIS_MODULE`, `.of_node` (or `.fwnode`), `.nr_resets`, and either `.of_xlate` (DT) or `.fwnode_xlate` (firmware-neutral); calls `devm_reset_controller_register(dev, rcdev)` from probe.

REQ-2: DT binding (`Documentation/devicetree/bindings/reset/`): each controller node has `#reset-cells = <N>` where N defaults to 1 (line index); consumers reference `resets = <&rstc N>` (1-cell) or `<&rstc Ngroup Nidx>` (2-cell for grouped controllers).

REQ-3: `reset_controller_register` validates: cannot set both `of_node` and `fwnode` directly (one synthesized from the other); cannot set both `of_xlate` and `fwnode_xlate`; if neither xlate provided, install `fwnode_reset_simple_xlate` with `fwnode_reset_n_cells = 1`.

REQ-4: Per-consumer handle's `rcdev` pointer is `__rcu` and dereferenced under per-handle SRCU. On controller unregister, the framework synchronises SRCU then frees the rcdev — handles outliving the controller observe `rstc->rcdev == NULL` and return `-ENODEV`.

REQ-5: **Exclusive** mode (default): exactly one `reset_control` may hold a given (rcdev, id); second exclusive get returns `-EBUSY`. **Shared** mode (`RESET_CONTROL_SHARED*`): multiple consumers share one handle; `deassert_count` ref-counts how many have called `_deassert`, hardware deassert only fires when count rises 0→1, hardware assert only fires when count drops 1→0.

REQ-6: **Acquired vs released** (orthogonal to shared/exclusive): only used for exclusive lines that need to transfer ownership; `_acquire()` claims the line, `_release()` lets a sibling consumer take it.

REQ-7: `reset_control_reset(rstc)` semantics:
- exclusive: refuse if `!rstc->acquired`; call `rcdev->ops->reset(...)`.
- shared: refuse if `deassert_count != 0`; atomic-inc `triggered_count`, only first caller (count 0→1) actually pulses; later calls return 0; `_rearm` decrements to allow next pulse.

REQ-8: `reset_control_assert(rstc)` semantics:
- shared: dec `deassert_count`; if it reaches 0, actually call `rcdev->ops->assert(...)`.
- exclusive: refuse if `!acquired`; refuse if `triggered_count != 0`; call `rcdev->ops->assert(...)`.

REQ-9: `reset_control_deassert(rstc)`:
- shared: inc `deassert_count`; if it rises to 1, call `rcdev->ops->deassert(...)`.
- exclusive: same WARN-checks as assert; call `rcdev->ops->deassert(...)`.

REQ-10: `reset_control_array` aggregates N child controls; `_reset/_assert/_deassert/_acquire/_release` iterate in order and roll back on failure (best-effort).

REQ-11: GPIO-reset adapter: when a consumer references `reset-gpios = <&gpio N flags>`, the core's `__reset_add_reset_gpio_device` looks up the GPIO provider, allocates a `reset_gpio_lookup`, registers an auxiliary device on the auxiliary bus with `name = "gpio"`, and a separate `reset-gpio` aux driver (in `drivers/reset/reset-gpio.c`) probes that aux device and registers a real `reset_controller_dev` backed by the GPIO toggle.

REQ-12: `try_module_get(rcdev->owner)` taken on every `__reset_control_get_internal`; ensures controller module cannot unload while consumers hold handles.

## Acceptance Criteria

- [ ] AC-1: Boot SoC with `reset-simple` controller (e.g. STM32, SoCFPGA, Allwinner sun6i) — `ls /sys/devices/.../reset` enumerates the controller; consumer probes succeed.
- [ ] AC-2: `reset_control_reset(rstc)` on a shared line fires hardware pulse exactly once when called from multiple consumers; subsequent calls return 0 without HW toggle.
- [ ] AC-3: Exclusive `_assert` then `_deassert` from a USB controller driver brings the USB block out of reset; consumer DT lookup resolves to the right line index.
- [ ] AC-4: Concurrent get-of-same-exclusive-line from two consumers returns `-EBUSY` on the second.
- [ ] AC-5: `reset_control_acquire` from consumer B succeeds after consumer A's `_release`; line state preserved.
- [ ] AC-6: Bulk API: probe a device with `resets = <&rstc 1>, <&rstc 2>, <&rstc 3>` and call `reset_control_bulk_deassert(3, rstcs)` — all three lines deassert in order; if line 3 fails, lines 1+2 re-assert.
- [ ] AC-7: GPIO-reset adapter: consumer with `reset-gpios = <&gpio0 5 GPIO_ACTIVE_LOW>` resolves through aux-bus to a synthesized rcdev; `_deassert` toggles the GPIO.
- [ ] AC-8: `rmmod` test (module-based controllers): controller unregister with outstanding consumer handles → SRCU sync; consumer `_assert` post-unregister returns `-ENODEV` without UAF.

## Architecture

`Control` (Rust analogue):

```
struct Control {
  rcdev:           RcuPtr<ControllerDev>,   // __rcu, SRCU-protected
  srcu:            SrcuStruct,
  list:            ListEntry,               // on rcdev.reset_control_head
  id:              u32,
  refcnt:          Kref,
  acquired:        bool,
  shared:          bool,
  array:           bool,
  deassert_count:  AtomicU32,
  triggered_count: AtomicU32,
  lock:            Mutex<()>,               // serializes acquire/release
}

struct ControllerDev {
  ops:                 &'static ResetControlOps,
  owner:               Module,
  dev:                 Option<DevicePtr>,
  fwnode:              Option<FwnodePtr>,
  of_node:             Option<DeviceNodePtr>,
  list:                ListEntry,           // on reset_controller_list
  reset_control_head:  Mutex<List<Control>>,
  nr_resets:           u32,
  of_xlate:            Option<OfXlateFn>,
  fwnode_xlate:        Option<FwnodeXlateFn>,
  of_reset_n_cells:    u32,
  fwnode_reset_n_cells: u32,
  lock:                Mutex<()>,
}
```

Two top-level mutexes:
- `reset_list_mutex` protects the global `reset_controller_list` and `reset_gpio_lookup_list`.
- per-`rcdev.lock` protects `rcdev->reset_control_head` and the per-control `kref`.

Per-handle SRCU `rstc->srcu` protects `rstc->rcdev` pointer against controller unregister: `_assert`/`_deassert`/etc. take `guard(srcu)(&rstc->srcu)` before deref'ing `rstc->rcdev`; unregister does `rcu_replace_pointer(..., NULL)` then `synchronize_srcu`.

Init / get path `__reset_control_get`:
1. Resolve DT/fwnode reference: `of_parse_phandle_with_args` (or `fwnode_property_get_reference_args`) on `resets` property at the requested index.
2. Find matching `rcdev` in `reset_controller_list` by fwnode equality (under `reset_list_mutex`).
3. Call `rcdev->of_xlate` or `rcdev->fwnode_xlate` to decode cells → line `id`.
4. If `id >= rcdev->nr_resets`: `-EINVAL`.
5. Take `rcdev->lock`; walk `reset_control_head` looking for existing `(rcdev, id)`:
   - exclusive + non-shared exclusive get: `-EBUSY`.
   - shared + existing-shared: `kref_get` + return.
   - exclusive + existing-exclusive-released + new-not-acquired: reuse.
6. Else allocate new `reset_control`, `init_srcu_struct`, `try_module_get(rcdev->owner)`, `rcu_assign_pointer(rstc->rcdev, rcdev)`, link to `reset_control_head`.

Hot-path `reset_control_deassert(rstc)`:
1. If null or IS_ERR: short-circuit.
2. If array: dispatch to `_array_deassert`.
3. `guard(srcu)(&rstc->srcu)` + `srcu_dereference(rstc->rcdev)`; if NULL, `-ENODEV`.
4. shared branch: `atomic_inc_return(&deassert_count)`; if newly == 1, call `rcdev->ops->deassert(rcdev, id)`; if HW fails, `atomic_dec`.
5. exclusive branch: WARN if `triggered_count != 0`; require `acquired`; call `rcdev->ops->deassert(...)`.

Array path `reset_control_array_deassert`:
1. For i in 0..num_rstcs: `_deassert(resets->rstc[i])`.
2. On failure of element k, loop k..0 calling `_assert` to undo; return err.

GPIO-reset aux bus path:
1. Consumer DT has `reset-gpios = <&gpioX N flags>` (but no matching `resets` phandle).
2. `__reset_add_reset_gpio_device(fwnode, args)` checks `reset_gpio_lookup_list` for existing match; if none, allocates `reset_gpio_lookup` + `ida_alloc(&reset_gpio_ida)` for unique id, registers `auxiliary_device` with `name = "gpio"` on parent = GPIO provider device.
3. `drivers/reset/reset-gpio.c` (separate file) binds via aux-bus, allocates `reset_controller_dev` whose `.ops` are GPIO-set-based, registers via `devm_reset_controller_register`.
4. Consumer's `__reset_control_get` then finds this synthesized rcdev by fwnode match.

Controller unregister `reset_controller_unregister`:
1. `guard(mutex)(&reset_list_mutex)`.
2. `list_del(&rcdev->list)`.
3. `guard(mutex)(&rcdev->lock)`.
4. For each `rstc` in `reset_control_head`: `rcu_replace_pointer(rstc->rcdev, NULL, true)`; remove from list.
5. Consumers later observe NULL deref-via-SRCU and return `-ENODEV`; their `reset_control_put` finishes cleanup.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `deassert_count_balance` | INVARIANT | `deassert_count == 0` iff line currently asserted in HW (for shared lines). |
| `triggered_count_balance` | INVARIANT | `triggered_count ∈ {0,1}` on shared lines. |
| `xlate_in_range` | OOB | xlate returns `id < rcdev->nr_resets` or an `IS_ERR`-encoded negative; consumer rejects out-of-range. |
| `rcu_srcu_paired` | RAII | every `srcu_dereference(rstc->rcdev)` happens inside a matching `guard(srcu)`. |
| `array_rollback_complete` | INVARIANT | on bulk failure at index k, indices 0..k-1 are reverted before return. |

### Layer 2: TLA+

`models/reset/concurrent_get_put.tla` (future): models concurrent `_get` + `_put` + `_unregister` against a controller; proves no UAF on `rstc->rcdev` and no leaked module reference.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `_get_internal` post: returned `rstc` has `rstc->rcdev == requested rcdev` and `rstc->id == requested id` | `__reset_control_get_internal` |
| `_deassert` post (shared): `deassert_count > 0` and last `ops->deassert` happened iff transition was 0→1 | `reset_control_deassert` |
| `_assert` post (shared): `deassert_count >= 0` and last `ops->assert` happened iff transition was 1→0 | `reset_control_assert` |
| `_release` post: subsequent `_acquire` by another exclusive consumer succeeds | `reset_control_acquire/release` |

### Layer 4: Verus/Creusot functional

Encoded: for a shared line, the number of `ops->deassert` calls equals the maximum value of `deassert_count` over the lifetime of the line — i.e. ref-count semantics match hardware semantics (line is electrically deasserted iff any consumer wants it deasserted).

## Hardening

- Per-handle SRCU + per-rcdev mutex eliminate UAF on controller-unregister-vs-consumer-deref races.
- `try_module_get(rcdev->owner)` per-get; `module_put` on `__reset_control_release` — controller module cannot unload while live handles exist.
- xlate functions defended by `WARN_ON(flags & ~(SHARED|ACQUIRED))` filter to reject malformed flag bits.
- Array path always rolls back partial transitions to avoid asymmetric assert/deassert state.
- `RESET_CONTROL_FLAGS_BIT_*` plus the public `enum reset_control_flags` mapping isolated in `include/linux/reset.h`; no external surface lets a consumer fabricate invalid flag combinations.
- Per-line `id < nr_resets` boundary enforced in both default xlate and per-driver xlates.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `reset_control`, `reset_control_array` (variable-length VLA via `__counted_by`), and `reset_controller_dev` slab caches whitelisted; no direct userland copies anyway, but the slab classification is required for kmalloc-tracking.
- **PAX_KERNEXEC** — reset core text W^X; `reset_control_ops` vtable entries dispatched through kCFI; the framework's own dispatch helpers live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `reset_control_assert/_deassert/_reset` so consumer-IRQ-context callers don't expose stack-aligned gadgets.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-handle `kref` and on `deassert_count` / `triggered_count` (atomic_t today, but saturate-on-overflow expected on the Rookery port); overflow trap defeats hostile-driver assert/deassert imbalance UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `reset_control` and `reset_control_array` allocations so retired per-(rcdev,id) bindings cannot bleed to a successor `kzalloc_obj`.
- **PAX_UDEREF** — no direct userland surface; sysfs/debugfs reset-control state, if exposed, copies through canonical helpers with SMAP/PAN enforcement.
- **PAX_RAP / kCFI** — `reset_control_ops` (`.reset`/`.assert`/`.deassert`/`.status`), `of_xlate`/`fwnode_xlate`, and devres release callbacks marked kCFI-typed; refuse untagged indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `reset_controller_list`, per-rcdev base pointers, and per-handle `rcdev` pointers behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict reset-controller registration banners, xlate-failure warnings, and array-rollback messages to CAP_SYSLOG so an attacker cannot enumerate the SoC's reset topology from dmesg.
- **reset_control PAX_REFCOUNT** — `kref` overflow trap on the per-handle reference; defense against malicious consumer that calls `_get`/`_put` in unbalanced patterns to wrap the counter and trigger UAF.
- **of-parse args bounded** — `of_parse_phandle_with_args` validated against `#reset-cells`; xlate refuses cells outside the controller-declared range; defense against DT-shape confusion.
- **Exclusive vs shared lock** — `__reset_control_get_internal` strictly enforces `WARN_ON(!shared && !rstc->shared)` for second-time gets; defense against silent ownership escalation that would let an attacker steal a PHY hand-off.
- **GPIO-reset aux bus gate** — `reset_create_gpio_aux_device` allocates IDs from `reset_gpio_ida` bounded by IDA limits; refuse synthesized aux device when GPIO provider fwnode is not a registered GPIO controller.
- **Module owner pin** — `try_module_get` failure refuses get; defense against race between module unload and consumer probe.

Rationale: a reset line is a physical wire that can hold a peripheral block off the bus or release it onto it — flipping a reset line at the wrong moment crashes the SoC or, worse, lets an attacker-controlled DMA-capable block emit transactions before its firmware/config is loaded. kCFI on `reset_control_ops`, refcount-saturation on per-handle krefs, strict shared/exclusive enforcement on second-time gets, and module-owner pinning turn the reset core from a "best-effort ref-counted toggle" into a structurally enforced ownership boundary.

## Open Questions

- (none at this Tier-3 level)

## Out of Scope

- per-SoC controller drivers (covered in their own future Tier-3s under `.design/drivers/reset/`)
- `reset-simple.c` (covered in `reset-simple.md`)
- `reset-gpio.c` aux driver (future Tier-3)
- power-domain interactions (covered in `drivers/base/power/00-overview.md` future Tier-3)
- `Documentation/devicetree/bindings/reset/` schema details (covered per-binding in upstream docs)
- Implementation code
