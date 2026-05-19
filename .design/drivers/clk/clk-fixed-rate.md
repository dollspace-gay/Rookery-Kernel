# Tier-3: drivers/clk/clk-fixed-rate.c — Fixed-rate clock building block

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/clk/00-overview.md
upstream-paths:
  - drivers/clk/clk-fixed-rate.c (~238 lines)
  - include/linux/clk-provider.h (struct clk_fixed_rate, CLK_FIXED_RATE_PARENT_ACCURACY)
  - Documentation/devicetree/bindings/clock/fixed-clock.yaml (`fixed-clock` binding)
-->

## Summary

The **fixed-rate clock** is the simplest reusable hardware-block driver in the CCF (Common Clock Framework): it models a clock whose rate is a compile- or DT-time constant and which exposes neither gate nor rate-control. Per-block: `struct clk_fixed_rate` carries `fixed_rate` (Hz), `fixed_accuracy` (ppb), `flags`, and the `clk_hw` embedded so `container_of` recovers the wrapper. Per-rate: `recalc_rate` returns `fixed_rate` ignoring the parent; per-accuracy: `recalc_accuracy` returns `fixed_accuracy` *unless* `CLK_FIXED_RATE_PARENT_ACCURACY` is set, in which case the accuracy is inherited from the parent. The clock has no `prepare`, `enable`, `set_rate`, `round_rate`, or `set_parent` callbacks — `clk_prepare`/`clk_enable` walk only the parent chain. Registration is via the unified entry point `__clk_hw_register_fixed_rate()`, wrapped by `clk_register_fixed_rate()`, `clk_hw_register_fixed_rate()`, `clk_hw_register_fixed_rate_with_accuracy()`, and `devm_clk_hw_register_fixed_rate()`. A `CLK_OF_DECLARE` early-boot setup (`of_fixed_clk_setup`) and a deferred `builtin_platform_driver` (`of_fixed_clk_driver`) bind to DT nodes with `compatible = "fixed-clock"`, reading `clock-frequency` (mandatory u32 Hz), `clock-accuracy` (optional u32 ppb), and `clock-output-names` (optional string override of `node->name`). Critical for: oscillator / crystal references at the root of the clock tree, board-level fixed regulators-of-frequency, and providing the leaf "always-on, never-changing" anchor that mux / divider / gate blocks downcast to during rate-propagation.

This Tier-3 covers `drivers/clk/clk-fixed-rate.c` (~238 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct clk_fixed_rate` | per-block HW data | `ClkFixedRate` |
| `clk_fixed_rate_ops` | per-vtable | `CLK_FIXED_RATE_OPS` |
| `to_clk_fixed_rate(hw)` | per-`container_of` cast | `ClkFixedRate::from_hw` |
| `clk_fixed_rate_recalc_rate()` | per-rate-readback | `ClkFixedRate::recalc_rate` |
| `clk_fixed_rate_recalc_accuracy()` | per-accuracy-readback | `ClkFixedRate::recalc_accuracy` |
| `__clk_hw_register_fixed_rate()` | per-unified register | `ClkFixedRate::hw_register_inner` |
| `clk_register_fixed_rate()` | per-legacy `struct clk *` register | `ClkFixedRate::register_legacy` |
| `clk_hw_register_fixed_rate()` (inline) | per-`clk_hw *` register (no accuracy) | `ClkFixedRate::hw_register` |
| `clk_hw_register_fixed_rate_with_accuracy()` (inline) | per-`clk_hw *` register w/ accuracy | `ClkFixedRate::hw_register_with_accuracy` |
| `devm_clk_hw_register_fixed_rate()` (inline) | per-devres-managed register | `ClkFixedRate::devm_register` |
| `devm_clk_hw_register_fixed_rate_release()` | per-devres release callback | `ClkFixedRate::devm_release` |
| `clk_unregister_fixed_rate()` | per-legacy unregister + free | `ClkFixedRate::unregister_legacy` |
| `clk_hw_unregister_fixed_rate()` | per-`clk_hw *` unregister + free | `ClkFixedRate::hw_unregister` |
| `of_fixed_clk_setup()` | per-`CLK_OF_DECLARE` early boot | `ClkFixedRate::of_setup` |
| `_of_fixed_clk_setup()` | per-DT-parse helper | `ClkFixedRate::of_setup_inner` |
| `of_fixed_clk_probe()` | per-`platform_driver` deferred probe | `ClkFixedRate::of_probe` |
| `of_fixed_clk_remove()` | per-`platform_driver` remove | `ClkFixedRate::of_remove` |
| `of_fixed_clk_driver` | per-builtin platform driver | `OF_FIXED_CLK_DRIVER` |
| `of_fixed_clk_ids[]` | per-`of_device_id` match table | `OF_FIXED_CLK_IDS` |
| `CLK_FIXED_RATE_PARENT_ACCURACY` | per-flag: inherit parent accuracy | shared constant |

## Compatibility contract

REQ-1: struct clk_fixed_rate layout (must match `include/linux/clk-provider.h`):
- hw: embedded `struct clk_hw` (must be first field for `container_of` arithmetic? actually any position — `to_clk_fixed_rate` uses `container_of(_hw, struct clk_fixed_rate, hw)`).
- fixed_rate: u64 / `unsigned long` Hz (per-arch-pointer-width).
- fixed_accuracy: `unsigned long` ppb (parts-per-billion).
- flags: u8 — only `CLK_FIXED_RATE_PARENT_ACCURACY` is defined.

REQ-2: clk_fixed_rate_ops vtable:
- `.recalc_rate` = `clk_fixed_rate_recalc_rate`.
- `.recalc_accuracy` = `clk_fixed_rate_recalc_accuracy`.
- No other ops (no `.enable`, no `.disable`, no `.prepare`, no `.set_rate`, no `.round_rate`, no `.set_parent`, no `.determine_rate`).
- Exported `EXPORT_SYMBOL_GPL`.

REQ-3: clk_fixed_rate_recalc_rate(hw, parent_rate):
- fixed = to_clk_fixed_rate(hw).
- return fixed.fixed_rate.
- /* `parent_rate` is ignored — that is the whole point of "fixed". */

REQ-4: clk_fixed_rate_recalc_accuracy(hw, parent_accuracy):
- fixed = to_clk_fixed_rate(hw).
- if (fixed.flags & CLK_FIXED_RATE_PARENT_ACCURACY):
  - return parent_accuracy.
- return fixed.fixed_accuracy.

REQ-5: __clk_hw_register_fixed_rate(dev, np, name, parent_name, parent_hw, parent_data, flags, fixed_rate, fixed_accuracy, clk_fixed_flags, devm):
- /* Allocate */
- if devm:
  - fixed = devres_alloc(devm_clk_hw_register_fixed_rate_release, sizeof(*fixed), GFP_KERNEL).
- else:
  - fixed = kzalloc(sizeof(*fixed), GFP_KERNEL).
- if !fixed: return ERR_PTR(-ENOMEM).
- /* Build init descriptor */
- init.name = name.
- init.ops = &clk_fixed_rate_ops.
- init.flags = flags.
- init.parent_names = parent_name ? &parent_name : NULL.
- init.parent_hws = parent_hw ? &parent_hw : NULL.
- init.parent_data = parent_data.
- init.num_parents = (parent_name || parent_hw || parent_data) ? 1 : 0.
- /* Populate fixed-rate fields */
- fixed.flags = clk_fixed_flags.
- fixed.fixed_rate = fixed_rate.
- fixed.fixed_accuracy = fixed_accuracy.
- fixed.hw.init = &init.
- /* Register */
- hw = &fixed.hw.
- if (dev || !np): ret = clk_hw_register(dev, hw).
- else: ret = of_clk_hw_register(np, hw).
- if ret:
  - if devm: devres_free(fixed).
  - else: kfree(fixed).
  - hw = ERR_PTR(ret).
- else if devm:
  - devres_add(dev, fixed).
- return hw.

REQ-6: clk_register_fixed_rate(dev, name, parent_name, flags, fixed_rate):
- hw = clk_hw_register_fixed_rate_with_accuracy(dev, name, parent_name, flags, fixed_rate, 0).
- if IS_ERR(hw): return ERR_CAST(hw).
- return hw->clk.

REQ-7: clk_hw_register_fixed_rate / _with_accuracy / devm_clk_hw_register_fixed_rate (inlines in `include/linux/clk-provider.h`):
- All resolve to `__clk_hw_register_fixed_rate(...)` with the appropriate combination of `devm=false`/`true` and `fixed_accuracy=0` or caller-supplied.

REQ-8: devm_clk_hw_register_fixed_rate_release(dev, res):
- fix = res /* devres_alloc gave us a `struct clk_fixed_rate` directly */.
- clk_hw_unregister(&fix->hw).
- /* DO NOT call clk_hw_unregister_fixed_rate because that would kfree() the storage that devres also owns — double-free. */
- /* devres core will kfree(res) after this returns. */

REQ-9: clk_unregister_fixed_rate(clk):
- hw = __clk_get_hw(clk).
- if !hw: return.
- clk_unregister(clk).
- kfree(to_clk_fixed_rate(hw)).

REQ-10: clk_hw_unregister_fixed_rate(hw):
- fixed = to_clk_fixed_rate(hw).
- clk_hw_unregister(hw).
- kfree(fixed).

REQ-11: of_fixed_clk_setup(node) [CLK_OF_DECLARE early-boot path]:
- Call `_of_fixed_clk_setup(node)`; the return value is discarded — failures are silent because CLK_OF_DECLARE has no error channel.

REQ-12: _of_fixed_clk_setup(node):
- clk_name = node->name (default).
- if of_property_read_u32(node, "clock-frequency", &rate): return ERR_PTR(-EIO) /* mandatory */.
- accuracy = 0; of_property_read_u32(node, "clock-accuracy", &accuracy) /* optional, default 0 */.
- of_property_read_string(node, "clock-output-names", &clk_name) /* optional rename */.
- hw = clk_hw_register_fixed_rate_with_accuracy(NULL, clk_name, NULL, 0, rate, accuracy).
- if IS_ERR(hw): return hw.
- ret = of_clk_add_hw_provider(node, of_clk_hw_simple_get, hw).
- if ret: clk_hw_unregister_fixed_rate(hw); return ERR_PTR(ret).
- return hw.

REQ-13: of_fixed_clk_probe(pdev):
- /* Runs only if of_fixed_clk_setup didn't already register the clock (e.g. node parsed late, or modular build). */
- hw = _of_fixed_clk_setup(pdev->dev.of_node).
- if IS_ERR(hw): return PTR_ERR(hw).
- platform_set_drvdata(pdev, hw).
- return 0.

REQ-14: of_fixed_clk_remove(pdev):
- hw = platform_get_drvdata(pdev).
- of_clk_del_provider(pdev->dev.of_node).
- clk_hw_unregister_fixed_rate(hw).

REQ-15: of_fixed_clk_driver registration:
- of_fixed_clk_ids = [ { .compatible = "fixed-clock" }, { } ].
- of_fixed_clk_driver = { .driver = { .name = "of_fixed_clk", .of_match_table = of_fixed_clk_ids }, .probe = of_fixed_clk_probe, .remove = of_fixed_clk_remove }.
- builtin_platform_driver(of_fixed_clk_driver).

REQ-16: Devicetree binding `fixed-clock` (per Documentation/devicetree/bindings/clock/fixed-clock.yaml):
- compatible = "fixed-clock" (required).
- #clock-cells = <0> (required for `of_clk_hw_simple_get`).
- clock-frequency = <hz> (required, u32).
- clock-accuracy = <ppb> (optional, u32, default 0).
- clock-output-names = "name" (optional, renames the clock).

REQ-17: CLK_FIXED_RATE_PARENT_ACCURACY flag:
- Stored in `struct clk_fixed_rate::flags`.
- When set, `recalc_accuracy` ignores `fixed_accuracy` and forwards the parent's accuracy.
- Note: this is a `clk_fixed_flags` (driver-private), not a `clk_init_data::flags` (CCF-wide).

## Acceptance Criteria

- [ ] AC-1: `recalc_rate` returns `fixed_rate` and ignores `parent_rate`.
- [ ] AC-2: `recalc_accuracy` returns `fixed_accuracy` when `CLK_FIXED_RATE_PARENT_ACCURACY` is clear.
- [ ] AC-3: `recalc_accuracy` returns `parent_accuracy` when `CLK_FIXED_RATE_PARENT_ACCURACY` is set.
- [ ] AC-4: `clk_fixed_rate_ops` installs only `.recalc_rate` and `.recalc_accuracy` — `clk_enable`/`clk_set_rate`/`clk_set_parent` on this clock walk to the parent (or are no-ops) because the ops are absent.
- [ ] AC-5: `__clk_hw_register_fixed_rate` with `devm=false` allocates via `kzalloc` and frees via `kfree` on error.
- [ ] AC-6: `__clk_hw_register_fixed_rate` with `devm=true` allocates via `devres_alloc` and on success calls `devres_add` so the release callback runs at device teardown.
- [ ] AC-7: `devm_clk_hw_register_fixed_rate_release` calls `clk_hw_unregister` but NOT `kfree` (devres frees the storage) — double-free regression test passes.
- [ ] AC-8: `clk_unregister_fixed_rate` and `clk_hw_unregister_fixed_rate` both call `kfree(to_clk_fixed_rate(hw))` for the non-devm path.
- [ ] AC-9: `of_fixed_clk_setup` parses `clock-frequency` (mandatory) — missing property yields `-EIO`.
- [ ] AC-10: `of_fixed_clk_setup` parses `clock-accuracy` as optional with default 0.
- [ ] AC-11: `of_fixed_clk_setup` honors `clock-output-names` overriding `node->name`.
- [ ] AC-12: `of_fixed_clk_setup` registers `of_clk_hw_simple_get` as the DT clock provider (i.e. `#clock-cells = <0>` resolves to the single registered `clk_hw`).
- [ ] AC-13: `of_fixed_clk_probe` is a no-op when `of_fixed_clk_setup` already registered the clock at early boot.
- [ ] AC-14: `of_fixed_clk_remove` calls `of_clk_del_provider` before `clk_hw_unregister_fixed_rate` (provider-before-clock teardown).
- [ ] AC-15: `init.num_parents` is 0 when no parent of any flavor is supplied, 1 otherwise.

## Architecture

```
struct ClkFixedRate {
  hw: ClkHw,                          // embedded; container_of recovers ClkFixedRate
  fixed_rate: u64,                    // Hz
  fixed_accuracy: u64,                // ppb
  flags: u8,                          // CLK_FIXED_RATE_PARENT_ACCURACY
}
```

`ClkFixedRate::from_hw(hw: *const ClkHw) -> *const ClkFixedRate`:
1. container_of(hw, ClkFixedRate, hw).

`ClkFixedRate::recalc_rate(hw: *const ClkHw, _parent_rate: u64) -> u64`:
1. (*Self::from_hw(hw)).fixed_rate.

`ClkFixedRate::recalc_accuracy(hw: *const ClkHw, parent_accuracy: u64) -> u64`:
1. fixed = Self::from_hw(hw).
2. if (fixed.flags & CLK_FIXED_RATE_PARENT_ACCURACY) != 0: return parent_accuracy.
3. return fixed.fixed_accuracy.

`CLK_FIXED_RATE_OPS: ClkOps = ClkOps { recalc_rate: Some(...), recalc_accuracy: Some(...), ..ClkOps::ZERO }`.

`ClkFixedRate::hw_register_inner(args) -> Result<*mut ClkHw, Errno>`:
1. /* Allocate */
2. fixed = if devm { devres_alloc(Self::devm_release, ...) } else { kzalloc::<ClkFixedRate>() }.
3. if fixed.is_null(): return Err(ENOMEM).
4. /* Build clk_init_data on the stack */
5. init = ClkInitData {
     name,
     ops: &CLK_FIXED_RATE_OPS,
     flags,
     parent_names: parent_name.map(|n| slice::from_ref(&n)),
     parent_hws: parent_hw.map(|h| slice::from_ref(&h)),
     parent_data,
     num_parents: if parent_name.is_some() || parent_hw.is_some() || parent_data.is_some() { 1 } else { 0 },
   }.
6. /* Populate */
7. (*fixed).flags = clk_fixed_flags.
8. (*fixed).fixed_rate = fixed_rate.
9. (*fixed).fixed_accuracy = fixed_accuracy.
10. (*fixed).hw.init = &init.
11. /* Register */
12. hw = &mut (*fixed).hw.
13. ret = if dev.is_some() || np.is_none() { clk_hw_register(dev, hw) } else { of_clk_hw_register(np, hw) }.
14. if let Err(e) = ret:
    - if devm: devres_free(fixed); else: kfree(fixed).
    - return Err(e).
15. if devm: devres_add(dev, fixed).
16. return Ok(hw).

`ClkFixedRate::devm_release(dev, res)`:
1. fix = res as *mut ClkFixedRate.
2. clk_hw_unregister(&fix.hw).
3. /* devres core frees res */.

`ClkFixedRate::hw_unregister(hw)`:
1. fixed = Self::from_hw(hw).
2. clk_hw_unregister(hw).
3. kfree(fixed).

`ClkFixedRate::of_setup_inner(node) -> Result<*mut ClkHw, Errno>`:
1. clk_name = node.name.
2. rate = of_property_read_u32(node, "clock-frequency").ok_or(Errno::EIO)?.
3. accuracy = of_property_read_u32(node, "clock-accuracy").unwrap_or(0).
4. if let Some(n) = of_property_read_string(node, "clock-output-names"): clk_name = n.
5. hw = Self::hw_register_with_accuracy(NULL, clk_name, NULL, 0, rate, accuracy)?.
6. of_clk_add_hw_provider(node, of_clk_hw_simple_get, hw).map_err(|e| { Self::hw_unregister(hw); e })?.
7. return Ok(hw).

`ClkFixedRate::of_setup(node)` [CLK_OF_DECLARE]:
1. let _ = Self::of_setup_inner(node) /* discard error — early-boot path has no error channel */.

`ClkFixedRate::of_probe(pdev) -> Result<(), Errno>`:
1. hw = Self::of_setup_inner(pdev.dev.of_node)?.
2. platform_set_drvdata(pdev, hw).
3. return Ok(()).

`ClkFixedRate::of_remove(pdev)`:
1. hw = platform_get_drvdata(pdev).
2. of_clk_del_provider(pdev.dev.of_node).
3. Self::hw_unregister(hw).

`OF_FIXED_CLK_IDS: [OfDeviceId] = [ OfDeviceId { compatible: "fixed-clock" } ]`.

`OF_FIXED_CLK_DRIVER: PlatformDriver = PlatformDriver { name: "of_fixed_clk", of_match_table: &OF_FIXED_CLK_IDS, probe: ClkFixedRate::of_probe, remove: ClkFixedRate::of_remove }`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `recalc_rate_returns_fixed` | INVARIANT | per-`recalc_rate`: ∀hw,pr. result == (*from_hw(hw)).fixed_rate. |
| `recalc_accuracy_flag_obeyed` | INVARIANT | per-`recalc_accuracy`: flag-set ⟹ parent; flag-clear ⟹ fixed. |
| `ops_table_minimal` | INVARIANT | per-`CLK_FIXED_RATE_OPS`: only `recalc_rate` + `recalc_accuracy` populated. |
| `hw_register_alloc_or_err` | INVARIANT | per-`hw_register_inner`: returns `*mut ClkHw` on Ok, ENOMEM/EINVAL on Err — no UAF. |
| `devm_release_no_double_free` | INVARIANT | per-`devm_release`: calls `clk_hw_unregister` but NOT `kfree` — devres frees. |
| `init_num_parents_correct` | INVARIANT | per-`hw_register_inner`: num_parents ∈ {0,1} matching `parent_*.is_some()`. |
| `of_clock_frequency_required` | INVARIANT | per-`of_setup_inner`: missing `clock-frequency` ⟹ Err(EIO). |
| `of_remove_provider_before_clock` | ORDERING | per-`of_remove`: of_clk_del_provider precedes hw_unregister. |

### Layer 2: TLA+

`drivers/clk/clk-fixed-rate.tla`:
- Per-register / per-unregister / per-devm-release lifecycle FSM (states: Unallocated → Allocated → Registered → Unregistered).
- Per-`recalc_rate` and per-`recalc_accuracy` are total functions of state.
- Properties:
  - `safety_no_double_free` — per-devm-path: `kfree(fixed)` is never called from `devm_release` (devres owns the slab).
  - `safety_no_use_after_free` — per-`hw_unregister`: kfree(fixed) is the last operation; no subsequent dereference.
  - `safety_recalc_rate_pure` — per-`recalc_rate`: no side effects, no IO, no allocation.
  - `safety_recalc_accuracy_flag_correctness` — per-`recalc_accuracy`: flag-set iff returns parent_accuracy.
  - `safety_provider_lifecycle` — per-DT path: provider registered iff `_of_fixed_clk_setup` returned Ok; deleted in `of_fixed_clk_remove` before clock unregister.
  - `liveness_register_eventually_completes` — per-`hw_register_inner`: returns Ok-or-Err in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ClkFixedRate::recalc_rate` post: ret == self.fixed_rate | `ClkFixedRate::recalc_rate` |
| `ClkFixedRate::recalc_accuracy` post: ret == parent_accuracy if flag else self.fixed_accuracy | `ClkFixedRate::recalc_accuracy` |
| `ClkFixedRate::hw_register_inner` post: Ok ⟹ hw.init.ops == &CLK_FIXED_RATE_OPS | `ClkFixedRate::hw_register_inner` |
| `ClkFixedRate::hw_register_inner` post: Ok ⟹ (*fixed).flags == clk_fixed_flags | `ClkFixedRate::hw_register_inner` |
| `ClkFixedRate::devm_release` post: clk_hw_unregister called; storage not freed by us | `ClkFixedRate::devm_release` |
| `ClkFixedRate::hw_unregister` post: clk_hw_unregister precedes kfree | `ClkFixedRate::hw_unregister` |
| `ClkFixedRate::of_setup_inner` post: Ok ⟹ of_clk_hw_simple_get registered as provider | `ClkFixedRate::of_setup_inner` |
| `ClkFixedRate::of_remove` post: of_clk_del_provider precedes hw_unregister | `ClkFixedRate::of_remove` |

### Layer 4: Verus/Creusot functional

`Per-register → hw initialized with (fixed_rate, fixed_accuracy, clk_fixed_flags), ops = &clk_fixed_rate_ops, init.num_parents matches inputs; recalc_rate returns fixed_rate; recalc_accuracy honors CLK_FIXED_RATE_PARENT_ACCURACY; unregister frees exactly once (kfree non-devm, devres-once devm); of_fixed_clk_setup parses required clock-frequency + optional clock-accuracy/clock-output-names and installs of_clk_hw_simple_get as provider` semantic equivalence: per-Documentation/devicetree/bindings/clock/fixed-clock.yaml and per-`include/linux/clk-provider.h` API surface.

## Hardening

(Inherits row-1 features from `drivers/clk/00-overview.md` § Hardening.)

Fixed-rate reinforcement:

- **Per-`recalc_rate` is pure and total** — defense against per-clock-tree-divergence from side-effectful readback.
- **Per-`parent_rate` ignored explicitly** — defense against per-spurious dependency on upstream value (the whole point of "fixed").
- **Per-`CLK_FIXED_RATE_PARENT_ACCURACY` flag-checked once, no fallback** — defense against per-accuracy-source ambiguity.
- **Per-devm release path does NOT `kfree(fixed)`** — defense against per-devres double-free (mirrors upstream comment in `devm_clk_hw_register_fixed_rate_release`).
- **Per-allocation failure unwinds via `devres_free`/`kfree` symmetrically** — defense against per-register-failure leak.
- **Per-`of_fixed_clk_setup` early-boot path: silent on failure but the *register failure* still unwinds the allocation** — defense against per-DT-malformed leak.
- **Per-`clock-frequency` mandatory** — defense against per-zero-Hz fixed clock that would NaN-propagate through divider/mux children.
- **Per-`of_clk_add_hw_provider` failure unregisters the just-registered `clk_hw`** — defense against per-orphan-clk leak.
- **Per-`of_fixed_clk_remove` order: `of_clk_del_provider` THEN `clk_hw_unregister_fixed_rate`** — defense against per-provider-resolves-to-freed-clk race.
- **Per-`init.num_parents` derived from exactly-one-of {parent_name, parent_hw, parent_data}** — defense against per-CCF rejection / per-bogus parent linkage.
- **Per-`clk_fixed_rate_ops` exported `_GPL`** — defense against per-proprietary-module ABI leakage.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/clk/clk.c clock-core registration / rate-propagation (covered in `clk-core.md` Tier-3)
- drivers/clk/clkdev.c consumer lookup (covered separately if expanded)
- drivers/clk/clk-divider.c (covered in `clk-divider.md` Tier-3)
- drivers/clk/clk-gate.c (covered in `clk-gate.md` Tier-3)
- drivers/clk/clk-mux.c (covered in `clk-mux.md` Tier-3)
- of/base.c devicetree parsing primitives (covered separately if expanded)
- Vendor-specific fixed-rate wrappers (SoC-specific drivers)
- Implementation code
