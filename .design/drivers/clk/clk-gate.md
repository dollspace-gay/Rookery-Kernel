# Tier-3: drivers/clk/clk-gate.c — Generic clock gate building block

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/clk/00-overview.md
upstream-paths:
  - drivers/clk/clk-gate.c (~259 lines)
  - include/linux/clk-provider.h (struct clk_gate, CLK_GATE_* flags, clk_gate_ops)
-->

## Summary

The **clock gate** is a reusable hardware-block driver in the CCF (Common Clock Framework) that models an MMIO single-bit "enable" register field controlling whether a clock signal propagates downstream. Per-block: `struct clk_gate` carries `hw` (embedded `clk_hw`), `reg` (`void __iomem *`), `bit_idx` (u8 in [0..31], or [0..15] when `_HIWORD_MASK`), `flags` (u8), and `lock` (optional `spinlock_t *` shared across gates that touch the same register). Per-`endisable` flips `set := flag(CLK_GATE_SET_TO_DISABLE) XOR enable`, then performs a read-modify-write or (under `CLK_GATE_HIWORD_MASK`) a single atomic write with the top 16 bits set as the write-mask — no lock required for hiword-mask hardware because the write is atomic by-construction at the bus. Per-endian via `CLK_GATE_BIG_ENDIAN` (ioread32be / iowrite32be vs readl / writel). Per-ops: `clk_gate_ops` exposes `.enable` / `.disable` / `.is_enabled` only — rate and parent are inherited. Registration goes through `__clk_hw_register_gate()` with `clk_register_gate()`, `clk_hw_register_gate()`, and `__devm_clk_hw_register_gate()` wrappers; teardown via `clk_unregister_gate()`, `clk_hw_unregister_gate()`, and a devres release callback. Critical for: every SoC's per-peripheral clock-enable bit, fine-grained runtime power management, and ABI-compatible wrappers of vendor clock drivers that compose a gate atop divider / mux building blocks.

This Tier-3 covers `drivers/clk/clk-gate.c` (~259 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct clk_gate` | per-gate HW data | `ClkGate` |
| `clk_gate_ops` | per-vtable | `CLK_GATE_OPS` |
| `to_clk_gate(hw)` | per-`container_of` cast | `ClkGate::from_hw` |
| `clk_gate_readl()` | per-LE/BE register read | `ClkGate::readl` |
| `clk_gate_writel()` | per-LE/BE register write | `ClkGate::writel` |
| `clk_gate_endisable()` | per-RMW enable/disable core | `ClkGate::endisable` |
| `clk_gate_enable()` | per-`.enable` shim | `ClkGate::enable` |
| `clk_gate_disable()` | per-`.disable` shim | `ClkGate::disable` |
| `clk_gate_is_enabled()` | per-`.is_enabled` shim | `ClkGate::is_enabled` |
| `__clk_hw_register_gate()` | per-unified register | `ClkGate::hw_register_inner` |
| `clk_register_gate()` | per-legacy `struct clk *` register | `ClkGate::register_legacy` |
| `clk_hw_register_gate()` (inline) | per-`clk_hw *` register | `ClkGate::hw_register` |
| `__devm_clk_hw_register_gate()` | per-devres-managed register | `ClkGate::devm_hw_register_inner` |
| `devm_clk_hw_release_gate()` | per-devres release callback | `ClkGate::devm_release` |
| `clk_unregister_gate()` | per-legacy unregister + free | `ClkGate::unregister_legacy` |
| `clk_hw_unregister_gate()` | per-`clk_hw *` unregister + free | `ClkGate::hw_unregister` |
| `CLK_GATE_SET_TO_DISABLE` | per-flag: bit-polarity inversion | shared constant |
| `CLK_GATE_HIWORD_MASK` | per-flag: top-16-bits write-mask atomic | shared constant |
| `CLK_GATE_BIG_ENDIAN` | per-flag: big-endian MMIO | shared constant |

## Compatibility contract

REQ-1: struct clk_gate layout (must match `include/linux/clk-provider.h`):
- hw: embedded `struct clk_hw`.
- reg: `void __iomem *` — MMIO pointer to the gate register.
- bit_idx: u8 — bit position [0..31] (must be [0..15] when `CLK_GATE_HIWORD_MASK`).
- flags: u8 — bitwise OR of `CLK_GATE_*` driver flags.
- lock: `spinlock_t *` — optional, shared across sibling gates on the same register; NULL ⟹ caller asserts no concurrency on this register.

REQ-2: clk_gate_readl(gate) -> u32:
- if (gate.flags & CLK_GATE_BIG_ENDIAN): return ioread32be(gate.reg).
- else: return readl(gate.reg).

REQ-3: clk_gate_writel(gate, val):
- if (gate.flags & CLK_GATE_BIG_ENDIAN): iowrite32be(val, gate.reg).
- else: writel(val, gate.reg).

REQ-4: clk_gate_endisable(hw, enable) [core RMW]:
- gate = to_clk_gate(hw).
- /* Polarity normalization: set XOR enable */
- set = (gate.flags & CLK_GATE_SET_TO_DISABLE) ? 1 : 0.
- set ^= enable.
- /* Lock acquisition */
- if (gate.lock): spin_lock_irqsave(gate.lock, &flags).
- else: __acquire(gate.lock) /* sparse annotation only */.
- /* RMW vs hiword-mask atomic */
- if (gate.flags & CLK_GATE_HIWORD_MASK):
  - reg = BIT(gate.bit_idx + 16) /* write-mask in top half */.
  - if set: reg |= BIT(gate.bit_idx) /* data bit in bottom half */.
- else:
  - reg = clk_gate_readl(gate).
  - if set: reg |= BIT(gate.bit_idx).
  - else: reg &= ~BIT(gate.bit_idx).
- clk_gate_writel(gate, reg).
- /* Lock release */
- if (gate.lock): spin_unlock_irqrestore(gate.lock, flags).
- else: __release(gate.lock).

REQ-5: clk_gate_enable(hw) -> 0:
- clk_gate_endisable(hw, 1).
- return 0.

REQ-6: clk_gate_disable(hw):
- clk_gate_endisable(hw, 0).

REQ-7: clk_gate_is_enabled(hw) -> 0 | 1:
- gate = to_clk_gate(hw).
- reg = clk_gate_readl(gate).
- /* Apply polarity */
- if (gate.flags & CLK_GATE_SET_TO_DISABLE): reg ^= BIT(gate.bit_idx).
- reg &= BIT(gate.bit_idx).
- return reg ? 1 : 0.

REQ-8: clk_gate_ops vtable:
- `.enable` = `clk_gate_enable`.
- `.disable` = `clk_gate_disable`.
- `.is_enabled` = `clk_gate_is_enabled`.
- No rate/parent/prepare ops (gate inherits rate and parent).
- Exported `EXPORT_SYMBOL_GPL`.

REQ-9: __clk_hw_register_gate(dev, np, name, parent_name, parent_hw, parent_data, flags, reg, bit_idx, clk_gate_flags, lock):
- /* HIWORD_MASK validation */
- if (clk_gate_flags & CLK_GATE_HIWORD_MASK) ∧ bit_idx > 15: pr_err("gate bit exceeds LOWORD field"); return ERR_PTR(-EINVAL).
- /* Allocate */
- gate = kzalloc(sizeof(*gate), GFP_KERNEL).
- if !gate: return ERR_PTR(-ENOMEM).
- /* Build init descriptor */
- init.name = name.
- init.ops = &clk_gate_ops.
- init.flags = flags.
- init.parent_names = parent_name ? &parent_name : NULL.
- init.parent_hws = parent_hw ? &parent_hw : NULL.
- init.parent_data = parent_data.
- init.num_parents = (parent_name || parent_hw || parent_data) ? 1 : 0.
- /* Populate gate */
- gate.reg = reg.
- gate.bit_idx = bit_idx.
- gate.flags = clk_gate_flags.
- gate.lock = lock.
- gate.hw.init = &init.
- /* Register */
- hw = &gate.hw.
- if (dev || !np): ret = clk_hw_register(dev, hw).
- else if np: ret = of_clk_hw_register(np, hw).
- if ret: kfree(gate); hw = ERR_PTR(ret).
- return hw.

REQ-10: clk_register_gate(dev, name, parent_name, flags, reg, bit_idx, clk_gate_flags, lock):
- hw = clk_hw_register_gate(dev, name, parent_name, flags, reg, bit_idx, clk_gate_flags, lock).
- if IS_ERR(hw): return ERR_CAST(hw).
- return hw->clk.

REQ-11: clk_hw_register_gate (inline in `include/linux/clk-provider.h`):
- Resolves to `__clk_hw_register_gate(dev, NULL, name, parent_name, NULL, NULL, flags, reg, bit_idx, clk_gate_flags, lock)`.

REQ-12: clk_unregister_gate(clk):
- hw = __clk_get_hw(clk).
- if !hw: return.
- gate = to_clk_gate(hw).
- clk_unregister(clk).
- kfree(gate).

REQ-13: clk_hw_unregister_gate(hw):
- gate = to_clk_gate(hw).
- clk_hw_unregister(hw).
- kfree(gate).

REQ-14: devm_clk_hw_release_gate(dev, res):
- clk_hw_unregister_gate(*(struct clk_hw **)res).
- /* The devres-allocated slot holds a *pointer* to the `clk_hw`, not the `clk_gate` itself; `clk_hw_unregister_gate` performs the `kfree(gate)`. */

REQ-15: __devm_clk_hw_register_gate(dev, np, name, parent_name, parent_hw, parent_data, flags, reg, bit_idx, clk_gate_flags, lock):
- ptr = devres_alloc(devm_clk_hw_release_gate, sizeof(*ptr), GFP_KERNEL) /* ptr is `struct clk_hw **` slot */.
- if !ptr: return ERR_PTR(-ENOMEM).
- hw = __clk_hw_register_gate(dev, np, name, parent_name, parent_hw, parent_data, flags, reg, bit_idx, clk_gate_flags, lock).
- if !IS_ERR(hw):
  - *ptr = hw.
  - devres_add(dev, ptr).
- else:
  - devres_free(ptr).
- return hw.

REQ-16: CLK_GATE_SET_TO_DISABLE semantics:
- Default (flag clear): bit_idx=1 ⟹ clock enabled; bit_idx=0 ⟹ clock disabled.
- Flag set (inverted): bit_idx=1 ⟹ clock disabled; bit_idx=0 ⟹ clock enabled.
- Implemented as `set := flag XOR enable`.

REQ-17: CLK_GATE_HIWORD_MASK semantics (Rockchip / Allwinner-style):
- Register layout: bits [31:16] = write-mask; bits [15:0] = data.
- A write to `BIT(bit_idx+16) | (set ? BIT(bit_idx) : 0)` atomically sets-or-clears bit `bit_idx` of the data half without reading first.
- bit_idx MUST be ≤ 15 (validated at registration; `-EINVAL` on violation).
- No lock acquisition is required at the hardware level — but `gate.lock`, if supplied, is still taken to preserve consumer-visible serialization.

REQ-18: CLK_GATE_BIG_ENDIAN semantics:
- All readl/writel are replaced by ioread32be/iowrite32be.
- Bit-numbering of `bit_idx` is *logical* (LSB=0), the endianness affects only the byte order of the 32-bit MMIO transaction.

## Acceptance Criteria

- [ ] AC-1: `clk_gate_enable` writes the configured bit_idx with the right polarity (per `CLK_GATE_SET_TO_DISABLE`).
- [ ] AC-2: `clk_gate_disable` writes the inverse polarity from enable.
- [ ] AC-3: `clk_gate_is_enabled` returns 1 iff the hardware bit reads "enabled" given the polarity flag, else 0 — restricted to ∈ {0, 1}.
- [ ] AC-4: `clk_gate_endisable` with `CLK_GATE_HIWORD_MASK`: emits a single `clk_gate_writel` of `BIT(bit_idx+16) | (set ? BIT(bit_idx) : 0)` and performs no MMIO read.
- [ ] AC-5: `clk_gate_endisable` without `CLK_GATE_HIWORD_MASK`: emits read-then-modify-then-write to `gate.reg`.
- [ ] AC-6: `clk_gate_endisable` with `gate.lock != NULL`: takes `spin_lock_irqsave` before MMIO and releases on exit (including HIWORD path).
- [ ] AC-7: `clk_gate_endisable` with `gate.lock == NULL`: emits `__acquire`/`__release` sparse markers but no actual lock.
- [ ] AC-8: `clk_gate_readl` / `clk_gate_writel` use big-endian primitives iff `CLK_GATE_BIG_ENDIAN` is set.
- [ ] AC-9: `__clk_hw_register_gate` returns `ERR_PTR(-EINVAL)` when `CLK_GATE_HIWORD_MASK` is set and `bit_idx > 15`.
- [ ] AC-10: `__clk_hw_register_gate` allocates with `kzalloc`, frees with `kfree` on registration failure.
- [ ] AC-11: `__devm_clk_hw_register_gate` on success calls `devres_add`; on registration failure calls `devres_free` (no leak).
- [ ] AC-12: `devm_clk_hw_release_gate` calls `clk_hw_unregister_gate` which performs `kfree(gate)` exactly once.
- [ ] AC-13: `clk_unregister_gate` and `clk_hw_unregister_gate` both call `kfree(gate)` on the non-devm path.
- [ ] AC-14: `init.num_parents` is 0 with no parent supplied, 1 otherwise; `init.ops == &clk_gate_ops` always.
- [ ] AC-15: `clk_gate_ops` exposes only `.enable`, `.disable`, `.is_enabled` — no `.prepare`/`.set_rate`/`.set_parent`.

## Architecture

```
struct ClkGate {
  hw: ClkHw,                          // embedded; container_of recovers ClkGate
  reg: *mut u8,                       // void __iomem *
  bit_idx: u8,                        // 0..31 (0..15 if HIWORD_MASK)
  flags: u8,                          // CLK_GATE_* OR-mask
  lock: Option<*mut SpinLock>,        // shared across siblings on same reg
}
```

`ClkGate::from_hw(hw: *const ClkHw) -> *const ClkGate`:
1. container_of(hw, ClkGate, hw).

`ClkGate::readl(&self) -> u32`:
1. if (self.flags & CLK_GATE_BIG_ENDIAN) != 0: return ioread32be(self.reg).
2. else: return readl(self.reg).

`ClkGate::writel(&self, val: u32)`:
1. if (self.flags & CLK_GATE_BIG_ENDIAN) != 0: iowrite32be(val, self.reg).
2. else: writel(val, self.reg).

`ClkGate::endisable(hw: *const ClkHw, enable: i32)`:
1. gate = Self::from_hw(hw).
2. set = if (gate.flags & CLK_GATE_SET_TO_DISABLE) != 0 { 1 } else { 0 }.
3. set ^= enable.
4. /* Lock */
5. guard = if let Some(l) = gate.lock { Some(spin_lock_irqsave(l)) } else { None /* __acquire marker */ }.
6. /* RMW vs HIWORD_MASK */
7. if (gate.flags & CLK_GATE_HIWORD_MASK) != 0:
   - reg = BIT(gate.bit_idx + 16) as u32.
   - if set != 0: reg |= BIT(gate.bit_idx) as u32.
8. else:
   - reg = gate.readl().
   - if set != 0: reg |= BIT(gate.bit_idx) as u32.
   - else: reg &= !BIT(gate.bit_idx) as u32.
9. gate.writel(reg).
10. /* Unlock on guard drop */.

`ClkGate::enable(hw) -> Result<(), Errno>`:
1. Self::endisable(hw, 1).
2. Ok(()).

`ClkGate::disable(hw)`:
1. Self::endisable(hw, 0).

`ClkGate::is_enabled(hw) -> bool`:
1. gate = Self::from_hw(hw).
2. reg = gate.readl().
3. if (gate.flags & CLK_GATE_SET_TO_DISABLE) != 0: reg ^= BIT(gate.bit_idx) as u32.
4. reg &= BIT(gate.bit_idx) as u32.
5. return reg != 0.

`CLK_GATE_OPS: ClkOps = ClkOps { enable: Some(...), disable: Some(...), is_enabled: Some(...), ..ClkOps::ZERO }`.

`ClkGate::hw_register_inner(args) -> Result<*mut ClkHw, Errno>`:
1. /* HIWORD_MASK bit_idx validation */
2. if (clk_gate_flags & CLK_GATE_HIWORD_MASK) != 0 && bit_idx > 15:
   - pr_err!("gate bit exceeds LOWORD field").
   - return Err(EINVAL).
3. gate = kzalloc::<ClkGate>().
4. if gate.is_null(): return Err(ENOMEM).
5. /* Build clk_init_data on stack */
6. init = ClkInitData {
     name,
     ops: &CLK_GATE_OPS,
     flags,
     parent_names: parent_name.map(|n| slice::from_ref(&n)),
     parent_hws: parent_hw.map(|h| slice::from_ref(&h)),
     parent_data,
     num_parents: if parent_name.is_some() || parent_hw.is_some() || parent_data.is_some() { 1 } else { 0 },
   }.
7. /* Populate */
8. (*gate).reg = reg.
9. (*gate).bit_idx = bit_idx.
10. (*gate).flags = clk_gate_flags.
11. (*gate).lock = lock.
12. (*gate).hw.init = &init.
13. hw = &mut (*gate).hw.
14. ret = if dev.is_some() || np.is_none() { clk_hw_register(dev, hw) } else { of_clk_hw_register(np, hw) }.
15. if let Err(e) = ret: kfree(gate); return Err(e).
16. return Ok(hw).

`ClkGate::devm_hw_register_inner(args) -> Result<*mut ClkHw, Errno>`:
1. ptr = devres_alloc::<*mut ClkHw>(Self::devm_release).
2. if ptr.is_null(): return Err(ENOMEM).
3. hw = Self::hw_register_inner(args).
4. match hw:
   - Ok(h): *ptr = h; devres_add(dev, ptr); Ok(h).
   - Err(e): devres_free(ptr); Err(e).

`ClkGate::devm_release(dev, res)`:
1. hw_ptr = *(res as *mut *mut ClkHw).
2. Self::hw_unregister(hw_ptr) /* this kfrees the gate */.

`ClkGate::hw_unregister(hw)`:
1. gate = Self::from_hw(hw).
2. clk_hw_unregister(hw).
3. kfree(gate).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `endisable_polarity_xor` | INVARIANT | per-`endisable`: written bit-state == (CLK_GATE_SET_TO_DISABLE ? 1 : 0) XOR enable. |
| `is_enabled_returns_bool` | INVARIANT | per-`is_enabled`: ret ∈ {0, 1}. |
| `hiword_mask_no_read` | INVARIANT | per-`endisable` with HIWORD_MASK: zero MMIO reads, exactly one MMIO write. |
| `hiword_mask_bit_idx_bound` | INVARIANT | per-`hw_register_inner`: HIWORD_MASK ⟹ bit_idx ≤ 15 else EINVAL. |
| `lock_taken_iff_present` | INVARIANT | per-`endisable`: gate.lock != NULL ⟹ spin_lock_irqsave taken-and-released; lock == NULL ⟹ neither. |
| `big_endian_uses_be_io` | INVARIANT | per-`readl`/`writel`: CLK_GATE_BIG_ENDIAN ⟹ ioread32be/iowrite32be. |
| `hw_register_alloc_or_err` | INVARIANT | per-`hw_register_inner`: returns Ok(hw) or Err(ENOMEM|EINVAL), no UAF on failure. |
| `devm_release_single_free` | INVARIANT | per-`devm_release`: kfree(gate) called exactly once via hw_unregister. |
| `init_num_parents_correct` | INVARIANT | per-`hw_register_inner`: num_parents ∈ {0,1} matching parent_*.is_some(). |
| `bit_idx_in_range` | INVARIANT | per-`endisable`: bit_idx ∈ [0..31] for non-HIWORD, [0..15] for HIWORD. |

### Layer 2: TLA+

`drivers/clk/clk-gate.tla`:
- Per-`endisable` state machine: read (or skip) → modify → write, with optional spinlock-protected section.
- Per-register-lifecycle FSM: Unallocated → Allocated → Registered → Unregistered.
- Properties:
  - `safety_enable_disable_consistency` — per-pair: enable-then-disable returns hardware bit to disabled state regardless of CLK_GATE_SET_TO_DISABLE polarity.
  - `safety_hiword_no_rmw_race` — per-HIWORD path: writes are atomic at the bus; no read-modify-write window observable to a concurrent writer of another bit on the same register.
  - `safety_rmw_lock_protected` — per-non-HIWORD path with `gate.lock`: read and write are linearizable under the lock; another gate sharing the lock observes consistent state.
  - `safety_polarity_xor_correctness` — per-`endisable`: write-bit == (flag XOR enable).
  - `safety_is_enabled_polarity` — per-`is_enabled`: ret iff hardware bit == "enabled" given polarity flag.
  - `safety_no_use_after_free` — per-`hw_unregister`: kfree(gate) is the final action; no subsequent dereference.
  - `safety_devm_single_free` — per-devm: `devres_add(ptr)` on Ok; `devres_free(ptr)` on Err; no double-free, no leak.
  - `liveness_endisable_terminates` — per-`endisable`: returns in finite steps (single MMIO op under lock).
  - `liveness_register_eventually_completes` — per-`hw_register_inner`: returns Ok-or-Err in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ClkGate::endisable` post: written value has bit `bit_idx` == (flag XOR enable) | `ClkGate::endisable` |
| `ClkGate::endisable` HIWORD_MASK post: write value == BIT(bit_idx+16) \| (set ? BIT(bit_idx) : 0) | `ClkGate::endisable` |
| `ClkGate::enable` post: ret == Ok(()) ∧ hardware bit is "enabled" | `ClkGate::enable` |
| `ClkGate::disable` post: hardware bit is "disabled" | `ClkGate::disable` |
| `ClkGate::is_enabled` post: ret iff hardware bit reads "enabled" given polarity | `ClkGate::is_enabled` |
| `ClkGate::readl` post: byte order matches CLK_GATE_BIG_ENDIAN | `ClkGate::readl` |
| `ClkGate::writel` post: byte order matches CLK_GATE_BIG_ENDIAN | `ClkGate::writel` |
| `ClkGate::hw_register_inner` post: Ok ⟹ hw.init.ops == &CLK_GATE_OPS, gate.flags == clk_gate_flags, gate.bit_idx == bit_idx | `ClkGate::hw_register_inner` |
| `ClkGate::hw_register_inner` HIWORD_MASK pre: bit_idx ≤ 15 else Err(EINVAL) | `ClkGate::hw_register_inner` |
| `ClkGate::devm_release` post: hw_unregister called once; kfree(gate) once | `ClkGate::devm_release` |
| `ClkGate::hw_unregister` post: clk_hw_unregister precedes kfree | `ClkGate::hw_unregister` |

### Layer 4: Verus/Creusot functional

`Per-register → gate initialized with (reg, bit_idx, clk_gate_flags, lock), ops = &clk_gate_ops, init.num_parents matches inputs; per-enable/disable, hardware bit bit_idx is set to (CLK_GATE_SET_TO_DISABLE_flag XOR enable) under appropriate lock (or by HIWORD-atomic write); per-is_enabled, returns boolean of hardware-state interpreted by polarity flag; per-unregister, gate is freed exactly once (kfree non-devm, hw_unregister-via-devres-release on devm)` semantic equivalence: per-`include/linux/clk-provider.h` API surface and per-`Documentation/driver-api/clk.rst`.

## Hardening

(Inherits row-1 features from `drivers/clk/00-overview.md` § Hardening.)

Clock-gate reinforcement:

- **Per-`bit_idx` HIWORD_MASK bound checked at register time** — defense against per-stray-write into the write-mask half of a hiword register.
- **Per-`endisable` polarity is XOR-normalized once** — defense against per-CLK_GATE_SET_TO_DISABLE inversion bugs across enable/disable/is_enabled.
- **Per-HIWORD path emits zero MMIO reads** — defense against per-read-side-effect on registers that latch on read or that expose stale-data semantics.
- **Per-non-HIWORD path under `gate.lock` when supplied** — defense against per-sibling-gate-RMW race on the same register (multiple gates sharing one MMIO word).
- **Per-`gate.lock` is taken `_irqsave`** — defense against per-IRQ-context re-entry deadlock when called from atomic contexts (typical for `.enable`/`.disable`).
- **Per-`__acquire`/`__release` sparse annotations even when `gate.lock` is NULL** — defense against per-static-analysis false-positive (preserves locking-context invariants in callers).
- **Per-`CLK_GATE_BIG_ENDIAN` honored uniformly across read and write** — defense against per-half-endian-swapped corruption.
- **Per-`is_enabled` masks back to a single bit and returns ∈ {0, 1}** — defense against per-CCF-misinterpretation of multi-bit hardware state.
- **Per-`__clk_hw_register_gate` failure unwinds with `kfree(gate)` (non-devm path)** — defense against per-register-failure leak.
- **Per-`__devm_clk_hw_register_gate` failure unwinds with `devres_free(ptr)` (devm path)** — defense against per-devres-stub leak.
- **Per-`devm_clk_hw_release_gate` calls `clk_hw_unregister_gate` which performs the single `kfree(gate)`** — defense against per-double-free (mirrors upstream invariant: devres holds the `clk_hw **` slot, gate is freed by the unregister helper).
- **Per-`clk_gate_ops` exported `_GPL`** — defense against per-proprietary-module ABI leakage.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on debugfs gate state reads.
- **PAX_KERNEXEC** — W^X enforcement on gate `enable` / `disable` / `is_enabled` callbacks.
- **PAX_RANDKSTACK** — kernel-stack randomization on gate `enable_lock`-protected entries.
- **PAX_REFCOUNT** — saturating refcount inherited from `clk_core` enable_count.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `clk_gate` slab (holds MMIO base + bit_idx + lock pointer).
- **PAX_UDEREF** — SMAP/SMEP enforcement on debugfs writer poking gate enable state.
- **PAX_RAP / kCFI** — `clk_gate_ops` vtable hardened against indirect-call hijack; per-gate ops `static const`.
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding in clk_summary enable column.
- **GRKERNSEC_DMESG** — syslog restriction on gate WARN (bit_idx out of range, register-mismatch).
- **PAX_CONSTIFY_PLUGIN** — `clk_gate_ops` literal `static const`.
- **CAP_SYS_ADMIN strict** — debugfs gate-toggle write gated by GR-RBAC.
- **PAX_SIZE_OVERFLOW** — `bit_idx` + `clk_gate_flags` arithmetic checked at register.
- **GRKERNSEC_SYSCTL** — gate-related boot-locked sysctls.
- **LSM `security_locked_down(LOCKDOWN_DEBUGFS)`** — denies debugfs gate write under integrity lockdown.

Per-doc rationale: clock gates are the single bit that powers (or unpowers) every IP block; an attacker who can flip a gate bit can disable security-critical peripherals (TRNG, IOMMU, watchdog) or enable disabled blocks behind the OS's back. PAX_RAP locks the `clk_gate_ops` vtable that every `enable`/`disable` indirects through, CAP_SYS_ADMIN + LSM lockdown gate the debugfs path, and PAX_MEMORY_SANITIZE wipes the cached MMIO base in `clk_gate` slab on free.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/clk/clk.c clock-core registration / enable-refcount (covered in `clk-core.md` Tier-3)
- drivers/clk/clkdev.c consumer lookup (covered separately if expanded)
- drivers/clk/clk-divider.c (covered in `clk-divider.md` Tier-3)
- drivers/clk/clk-fixed-rate.c (covered in `clk-fixed-rate.md` Tier-3)
- drivers/clk/clk-mux.c (covered in `clk-mux.md` Tier-3)
- drivers/base/devres.c devres core (covered separately if expanded)
- Vendor-specific gate wrappers (SoC-specific drivers)
- Implementation code
