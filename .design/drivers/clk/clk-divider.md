# Tier-3: drivers/clk/clk-divider.c — Generic adjustable clock divider

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/clk/00-overview.md
upstream-paths:
  - drivers/clk/clk-divider.c (~630 lines)
  - include/linux/clk-provider.h (struct clk_divider, struct clk_div_table, CLK_DIVIDER_* flags)
-->

## Summary

The generic **clock divider** is a reusable hardware-block driver: it models an MMIO bit-field that divides a parent clock by a runtime-programmable factor (`out = ceil(parent / div)`). Per-block: `struct clk_divider` (reg + shift + width + flags + optional `clk_div_table` + optional `spinlock_t`). Per-rate: `recalc_rate` reads the bit-field and decodes it, `determine_rate` scans candidate divisors picking the best per `CLK_DIVIDER_ROUND_CLOSEST` policy, `set_rate` encodes the chosen divisor and writes the register (optionally under provider lock, optionally as a hiword-mask atomic-write). Per-encoding policy is determined by mutually-exclusive flags: `_ONE_BASED`, `_POWER_OF_TWO`, `_EVEN_INTEGERS`, `_MAX_AT_ZERO`, table-driven, or default (`val = div - 1`). Per-bus-endian via `_BIG_ENDIAN`. Per-RO via `_READ_ONLY` (no `set_rate` op installed). Per-zero-divisor handled via `_ALLOW_ZERO`. Per-hiword-mask atomic-RMW via `_HIWORD_MASK` (Rockchip-style: top-16-bits = write-mask, bottom-16-bits = data). Critical for: SoC clock-tree provisioning, ABI-compatible re-implementation of vendor clock drivers built atop the generic divider primitive.

This Tier-3 covers `drivers/clk/clk-divider.c` (~630 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct clk_divider` | per-divider HW data | `ClkDivider` |
| `struct clk_div_table` | per-`{val, div}` entry | `ClkDivTable` |
| `clk_divider_ops` | per-RW vtable | `CLK_DIVIDER_OPS` |
| `clk_divider_ro_ops` | per-RO vtable | `CLK_DIVIDER_RO_OPS` |
| `to_clk_divider()` | per-`container_of` cast | `ClkDivider::from_hw` |
| `clk_div_readl()` | per-LE/BE register read | `ClkDivider::readl` |
| `clk_div_writel()` | per-LE/BE register write | `ClkDivider::writel` |
| `clk_div_mask(width)` | per-`(1<<width)-1` | `clk_div_mask` |
| `_get_maxdiv()` | per-flags max divisor | `ClkDivider::get_maxdiv` |
| `_get_div()` | per-val ⟶ div decode | `ClkDivider::get_div` |
| `_get_val()` | per-div ⟶ val encode | `ClkDivider::get_val` |
| `_get_table_div()` / `_get_table_val()` | per-table-search decode/encode | `ClkDivTable::lookup_div` / `lookup_val` |
| `_is_valid_div()` | per-flags validity check | `ClkDivider::is_valid_div` |
| `_round_up_table()` / `_round_down_table()` | per-table-rounding | `ClkDivTable::round_up` / `round_down` |
| `_div_round_up()` | per-default round-up | `ClkDivider::div_round_up` |
| `_div_round_closest()` | per-closest-of-up/down | `ClkDivider::div_round_closest` |
| `_div_round()` | per-flag dispatch | `ClkDivider::div_round` |
| `_is_best_div()` | per-search predicate | `ClkDivider::is_best_div` |
| `_next_div()` | per-iter step (linear/PoT/table) | `ClkDivider::next_div` |
| `clk_divider_bestdiv()` | per-determine_rate sweep | `ClkDivider::bestdiv` |
| `divider_recalc_rate()` | per-recalc helper (exported) | `divider_recalc_rate` |
| `divider_determine_rate()` | per-RW determine (exported) | `divider_determine_rate` |
| `divider_ro_determine_rate()` | per-RO determine (exported) | `divider_ro_determine_rate` |
| `divider_get_val()` | per-set encode helper (exported) | `divider_get_val` |
| `clk_divider_recalc_rate()` | per-`recalc_rate` op | `ClkDivider::recalc_rate` |
| `clk_divider_determine_rate()` | per-`determine_rate` op | `ClkDivider::determine_rate` |
| `clk_divider_set_rate()` | per-`set_rate` op | `ClkDivider::set_rate` |
| `__clk_hw_register_divider()` | per-`clk_hw` registration core | `ClkDivider::hw_register` |
| `__devm_clk_hw_register_divider()` | per-devres-managed registration | `ClkDivider::devm_hw_register` |
| `clk_register_divider_table()` | per-table-based legacy `clk*` API | `ClkDivider::register_table` |
| `clk_unregister_divider()` / `clk_hw_unregister_divider()` | per-teardown | `ClkDivider::unregister` / `hw_unregister` |
| `CLK_DIVIDER_ONE_BASED` | per-encoding `val == div` | flag bit |
| `CLK_DIVIDER_POWER_OF_TWO` | per-encoding `div == 1<<val` | flag bit |
| `CLK_DIVIDER_ALLOW_ZERO` | per-tolerate-zero-divisor | flag bit |
| `CLK_DIVIDER_HIWORD_MASK` | per-atomic-RMW write style | flag bit |
| `CLK_DIVIDER_ROUND_CLOSEST` | per-round-to-nearest policy | flag bit |
| `CLK_DIVIDER_READ_ONLY` | per-skip-`set_rate` | flag bit |
| `CLK_DIVIDER_MAX_AT_ZERO` | per-`val==0 ⟹ div = mask+1` | flag bit |
| `CLK_DIVIDER_EVEN_INTEGERS` | per-encoding `div = 2*(val+1)` | flag bit |
| `CLK_DIVIDER_BIG_ENDIAN` | per-BE register access | flag bit |

## Compatibility contract

REQ-1: struct clk_divider:
- hw: per-`struct clk_hw` (embedded).
- reg: per-`void __iomem *` MMIO register.
- shift: per-bit-field LSB position in register (u8).
- width: per-bit-field width in bits (u8).
- flags: per-`CLK_DIVIDER_*` mask (u8 effective; carried as `unsigned long`).
- table: per-optional `const struct clk_div_table *` mapping `{val, div}`; sentinel = `{.div = 0}`.
- lock: per-optional `spinlock_t *` shared with sibling clocks on the same register.

REQ-2: struct clk_div_table:
- val: per-register bit-field value (unsigned).
- div: per-runtime divisor (unsigned, 0 = sentinel).
- /* Table is a flat array terminated by `{.div = 0}`. */

REQ-3: Per-flag encoding (mutually exclusive families; checked in code order in `_get_div`/`_get_val`/`_get_maxdiv`):
- `CLK_DIVIDER_ONE_BASED`: `val == div`; maxdiv = `clk_div_mask(width)`.
- `CLK_DIVIDER_POWER_OF_TWO`: `div == 1 << val`; maxdiv = `1 << clk_div_mask(width)`.
- `CLK_DIVIDER_MAX_AT_ZERO`: `val == 0 ⟹ div == clk_div_mask(width) + 1`; otherwise `div == val`.
- `CLK_DIVIDER_EVEN_INTEGERS`: `div == 2 * (val + 1)`; maxdiv = `2 * (clk_div_mask(width) + 1)`.
- table set: `_get_table_div(table, val)`; maxdiv = `_get_table_maxdiv(table, width)` (largest `div` whose `val ≤ mask`).
- default: `div == val + 1`; maxdiv = `clk_div_mask(width) + 1`.

REQ-4: clk_div_mask(width): returns `(1 << width) - 1` for width < 32; defined in `include/linux/clk-provider.h`.

REQ-5: clk_div_readl(div) / clk_div_writel(div, val):
- if `div.flags & CLK_DIVIDER_BIG_ENDIAN`: `ioread32be` / `iowrite32be`.
- else: `readl` / `writel`.

REQ-6: divider_recalc_rate(hw, parent_rate, val, table, flags, width):
- div = `_get_div(table, val, flags, width)`.
- if div == 0:
  - WARN if `!(flags & CLK_DIVIDER_ALLOW_ZERO)` with msg "%s: Zero divisor and CLK_DIVIDER_ALLOW_ZERO not set".
  - return parent_rate.
- return `DIV_ROUND_UP_ULL((u64)parent_rate, div)`.

REQ-7: clk_divider_recalc_rate(hw, parent_rate):
- div = to_clk_divider(hw).
- val = `clk_div_readl(div) >> div.shift` masked by `clk_div_mask(div.width)`.
- return `divider_recalc_rate(hw, parent_rate, val, div.table, div.flags, div.width)`.

REQ-8: divider_get_val(rate, parent_rate, table, width, flags):
- div = `DIV_ROUND_UP_ULL((u64)parent_rate, rate)`.
- if `!_is_valid_div(table, div, flags)`: return -EINVAL.
- value = `_get_val(table, div, flags, width)`.
- return `min(value, clk_div_mask(width))`.

REQ-9: `_is_valid_div(table, div, flags)`:
- if `flags & CLK_DIVIDER_POWER_OF_TWO`: return `is_power_of_2(div)`.
- if table: return table contains `clkt.div == div`.
- else: return true.

REQ-10: clk_divider_bestdiv(hw, parent, rate, *best_parent_rate, table, width, flags):
- if rate == 0: rate = 1.
- maxdiv = `_get_maxdiv(table, width, flags)`.
- if !(`clk_hw_get_flags(hw) & CLK_SET_RATE_PARENT`):
  - parent_rate = *best_parent_rate.
  - bestdiv = `_div_round(table, parent_rate, rate, flags)`.
  - bestdiv = max(bestdiv, 1).
  - bestdiv = min(bestdiv, maxdiv).
  - return bestdiv.
- /* Otherwise sweep candidates and let parent round each `rate * i`. */
- maxdiv = min(ULONG_MAX / rate, maxdiv) — overflow clamp.
- for i = `_next_div(table, 0, flags)`; i ≤ maxdiv; i = `_next_div(table, i, flags)`:
  - if `rate * i == parent_rate_saved`: return i (ideal match; no parent re-rate needed).
  - parent_rate = `clk_hw_round_rate(parent, rate * i)`.
  - now = `DIV_ROUND_UP_ULL(parent_rate, i)`.
  - if `_is_best_div(rate, now, best, flags)`: bestdiv = i, best = now, *best_parent_rate = parent_rate.
- if bestdiv == 0: bestdiv = `_get_maxdiv(table, width, flags)`; *best_parent_rate = `clk_hw_round_rate(parent, 1)`.
- return bestdiv.

REQ-11: `_is_best_div(rate, now, best, flags)`:
- if `flags & CLK_DIVIDER_ROUND_CLOSEST`: return `abs(rate - now) < abs(rate - best)`.
- else: return `now ≤ rate ∧ now > best` (largest-rate-not-exceeding policy).

REQ-12: `_div_round(table, parent_rate, rate, flags)`:
- if `flags & CLK_DIVIDER_ROUND_CLOSEST`: `_div_round_closest`.
- else: `_div_round_up`.

REQ-13: `_div_round_up`:
- div = `DIV_ROUND_UP_ULL(parent_rate, rate)`.
- if `_POWER_OF_TWO`: div = `__roundup_pow_of_two(div)`.
- if table: div = `_round_up_table(table, div)`.

REQ-14: `_div_round_closest`:
- up = `DIV_ROUND_UP_ULL(parent_rate, rate)`; down = `parent_rate / rate`.
- if `_POWER_OF_TWO`: up = `__roundup_pow_of_two(up)`, down = `__rounddown_pow_of_two(down)`.
- elif table: up = `_round_up_table(table, up)`, down = `_round_down_table(table, down)`.
- up_rate = `DIV_ROUND_UP_ULL(parent_rate, up)`; down_rate = `DIV_ROUND_UP_ULL(parent_rate, down)`.
- return `(rate - up_rate) ≤ (down_rate - rate) ? up : down`.

REQ-15: divider_determine_rate(hw, req, table, width, flags):
- div = `clk_divider_bestdiv(hw, req.best_parent_hw, req.rate, &req.best_parent_rate, table, width, flags)`.
- req.rate = `DIV_ROUND_UP_ULL(req.best_parent_rate, div)`.
- return 0.

REQ-16: divider_ro_determine_rate(hw, req, table, width, flags, val):
- div = `_get_div(table, val, flags, width)`.
- if `CLK_SET_RATE_PARENT` set:
  - if !req.best_parent_hw: return -EINVAL.
  - req.best_parent_rate = `clk_hw_round_rate(req.best_parent_hw, req.rate * div)`.
- req.rate = `DIV_ROUND_UP_ULL(req.best_parent_rate, div)`.
- return 0.

REQ-17: clk_divider_determine_rate dispatch:
- if `div.flags & CLK_DIVIDER_READ_ONLY`:
  - val = read+shift+mask; call `divider_ro_determine_rate`.
- else: call `divider_determine_rate`.

REQ-18: clk_divider_set_rate(hw, rate, parent_rate):
- value = `divider_get_val(rate, parent_rate, div.table, div.width, div.flags)`.
- if value < 0: return value (propagate -EINVAL).
- if div.lock: `spin_lock_irqsave(div.lock, flags)` else `__acquire(div.lock)` (sparse annotation).
- if `div.flags & CLK_DIVIDER_HIWORD_MASK`:
  - val = `clk_div_mask(div.width) << (div.shift + 16)` (write-enable mask only; no read).
- else:
  - val = `clk_div_readl(div)`; val &= `~(clk_div_mask(div.width) << div.shift)`.
- val |= `(u32)value << div.shift`.
- `clk_div_writel(div, val)`.
- if div.lock: `spin_unlock_irqrestore` else `__release(div.lock)`.
- return 0.

REQ-19: clk_divider_ops vtable:
- .recalc_rate = clk_divider_recalc_rate.
- .determine_rate = clk_divider_determine_rate.
- .set_rate = clk_divider_set_rate.

REQ-20: clk_divider_ro_ops vtable:
- .recalc_rate = clk_divider_recalc_rate.
- .determine_rate = clk_divider_determine_rate.
- /* No .set_rate — writes prohibited. */

REQ-21: __clk_hw_register_divider(dev, np, name, parent_name, parent_hw, parent_data, flags, reg, shift, width, clk_divider_flags, table, lock):
- if `clk_divider_flags & CLK_DIVIDER_HIWORD_MASK` ∧ `width + shift > 16`: pr_warn("divider value exceeds LOWORD field"); return ERR_PTR(-EINVAL).
- div = kzalloc(sizeof(*div), GFP_KERNEL); if !div: return ERR_PTR(-ENOMEM).
- init.name = name.
- init.ops = `(clk_divider_flags & CLK_DIVIDER_READ_ONLY) ? &clk_divider_ro_ops : &clk_divider_ops`.
- init.flags = flags.
- init.parent_names = parent_name ? &parent_name : NULL.
- init.parent_hws = parent_hw ? &parent_hw : NULL.
- init.parent_data = parent_data.
- init.num_parents = (parent_name ∨ parent_hw ∨ parent_data) ? 1 : 0.
- Populate div→{reg, shift, width, flags, lock, table, hw.init}.
- hw = &div.hw; ret = `clk_hw_register(dev, hw)`.
- if ret: kfree(div); return ERR_PTR(ret).
- return hw.

REQ-22: __devm_clk_hw_register_divider: same as REQ-21 but devres-tracked via `devres_alloc(devm_clk_hw_release_divider, ...)`; on success `devres_add`; on failure `devres_free`.

REQ-23: clk_register_divider_table: legacy wrapper returning `struct clk *`; calls `__clk_hw_register_divider(..., NULL, NULL, NULL, flags, ...)` and returns `hw.clk` via `ERR_CAST` on error.

REQ-24: clk_unregister_divider(clk):
- hw = `__clk_get_hw(clk)`; if !hw: return.
- div = `to_clk_divider(hw)`.
- `clk_unregister(clk)`; kfree(div).

REQ-25: clk_hw_unregister_divider(hw):
- div = `to_clk_divider(hw)`.
- `clk_hw_unregister(hw)`; kfree(div).

REQ-26: Per-`devm_clk_hw_release_divider(dev, res)`:
- Unwrap stored `clk_hw **`; call `clk_hw_unregister_divider`.

REQ-27: Locking discipline:
- div.lock is OPTIONAL.
- If non-NULL: held over `read-modify-write` (and over the single hiword write).
- If NULL: caller asserts single-writer ownership; sparse `__acquire/__release` annotations balance the lock-required compile checks.
- All readl/writel inside lock when lock provided.

REQ-28: Per-CLK_DIVIDER_HIWORD_MASK semantics:
- Register layout: bits [31:16] are write-enable mask (set ⟹ bits [15:0] take effect on this write); bits [15:0] are data.
- Implies width + shift ≤ 16 (enforced at register time).
- Eliminates read-modify-write race; permits lockless concurrent updates of disjoint fields.
- Both `set_rate` and `set_parent` (mux) treat HIWORD_MASK the same way.

REQ-29: Per-CLK_SET_RATE_PARENT flag (consumer-defined, not divider-defined):
- If propagated, `bestdiv` sweeps candidates and asks the parent to round each (`rate * i`); divider chases parent rate.
- If not set, divider rounds within fixed parent.

## Acceptance Criteria

- [ ] AC-1: `clk_divider_recalc_rate` reads register, masks, and returns `ceil(parent / div)`.
- [ ] AC-2: Zero divisor with `_ALLOW_ZERO` set returns parent_rate without warning.
- [ ] AC-3: Zero divisor without `_ALLOW_ZERO` returns parent_rate and emits WARN_ON.
- [ ] AC-4: `_POWER_OF_TWO` encoding: `div == 1 << val` round-trips through `_get_div` / `_get_val`.
- [ ] AC-5: `_MAX_AT_ZERO`: register value 0 decodes to divisor `mask+1`; divisor `mask+1` encodes to value 0.
- [ ] AC-6: `_EVEN_INTEGERS`: divisor = `2*(val+1)`; round-trip preserved.
- [ ] AC-7: Table-based: arbitrary `{val, div}` pairs, sentinel `{0,0}`; unknown div returns 0 (invalid).
- [ ] AC-8: `_ROUND_CLOSEST`: picks nearer of round-up vs round-down divisor.
- [ ] AC-9: `_HIWORD_MASK`: `set_rate` writes `(mask << (shift+16)) | (val << shift)` without prior read.
- [ ] AC-10: `_HIWORD_MASK` + width+shift > 16: registration fails with -EINVAL.
- [ ] AC-11: `_READ_ONLY`: ops table is `clk_divider_ro_ops` (no `.set_rate`); `set_rate` not callable.
- [ ] AC-12: `_BIG_ENDIAN`: reads via `ioread32be`, writes via `iowrite32be`.
- [ ] AC-13: Provider lock held during read-modify-write when `div.lock != NULL`.
- [ ] AC-14: `__devm_clk_hw_register_divider` auto-unregisters on device unbind.
- [ ] AC-15: `clk_divider_bestdiv` with `CLK_SET_RATE_PARENT` clamps `maxdiv` to `ULONG_MAX / rate` to avoid overflow.

## Architecture

```
struct ClkDivider {
  hw: ClkHw,
  reg: *mut MmioReg,
  shift: u8,
  width: u8,
  flags: u32,                        // CLK_DIVIDER_*
  table: Option<&'static [ClkDivTable]>,
  lock: Option<*Spinlock>,
}

struct ClkDivTable {
  val: u32,
  div: u32,                          // 0 = sentinel
}

const CLK_DIVIDER_ONE_BASED:     u32 = 1 << 0;
const CLK_DIVIDER_POWER_OF_TWO:  u32 = 1 << 1;
const CLK_DIVIDER_ALLOW_ZERO:    u32 = 1 << 2;
const CLK_DIVIDER_HIWORD_MASK:   u32 = 1 << 3;
const CLK_DIVIDER_ROUND_CLOSEST: u32 = 1 << 4;
const CLK_DIVIDER_READ_ONLY:     u32 = 1 << 5;
const CLK_DIVIDER_MAX_AT_ZERO:   u32 = 1 << 6;
const CLK_DIVIDER_EVEN_INTEGERS: u32 = 1 << 7;
const CLK_DIVIDER_BIG_ENDIAN:    u32 = 1 << 8;
```

`ClkDivider::readl(&self) -> u32`:
1. if `self.flags & CLK_DIVIDER_BIG_ENDIAN`: return `ioread32be(self.reg)`.
2. else: return `readl(self.reg)`.

`ClkDivider::writel(&self, val: u32)`:
1. if `self.flags & CLK_DIVIDER_BIG_ENDIAN`: `iowrite32be(val, self.reg)`.
2. else: `writel(val, self.reg)`.

`ClkDivider::get_div(table, val, flags, width) -> u32`:
1. if `flags & CLK_DIVIDER_ONE_BASED`: return val.
2. if `flags & CLK_DIVIDER_POWER_OF_TWO`: return 1 << val.
3. if `flags & CLK_DIVIDER_MAX_AT_ZERO`: return val != 0 ? val : clk_div_mask(width) + 1.
4. if `flags & CLK_DIVIDER_EVEN_INTEGERS`: return 2 * (val + 1).
5. if table.is_some(): return table.lookup_div(val) (0 if not found).
6. return val + 1.

`ClkDivider::get_val(table, div, flags, width) -> u32`:
1. if `CLK_DIVIDER_ONE_BASED`: return div.
2. if `CLK_DIVIDER_POWER_OF_TWO`: return `__ffs(div)`.
3. if `CLK_DIVIDER_MAX_AT_ZERO`: return (div == clk_div_mask(width) + 1) ? 0 : div.
4. if `CLK_DIVIDER_EVEN_INTEGERS`: return (div >> 1) - 1.
5. if table: return table.lookup_val(div) (0 if not found).
6. return div - 1.

`ClkDivider::get_maxdiv(table, width, flags) -> u32`:
1. if `_ONE_BASED`: return clk_div_mask(width).
2. if `_POWER_OF_TWO`: return 1 << clk_div_mask(width).
3. if `_EVEN_INTEGERS`: return 2 * (clk_div_mask(width) + 1).
4. if table: return max(`clkt.div`) over `clkt.val ≤ mask`.
5. return clk_div_mask(width) + 1.

`divider_recalc_rate(hw, parent_rate, val, table, flags, width) -> u64`:
1. div = `ClkDivider::get_div(table, val, flags, width)`.
2. if div == 0:
   - WARN if `!(flags & CLK_DIVIDER_ALLOW_ZERO)`.
   - return parent_rate.
3. return `DIV_ROUND_UP_ULL(parent_rate, div)`.

`ClkDivider::recalc_rate(&self, parent_rate) -> u64`:
1. val = (self.readl() >> self.shift) & clk_div_mask(self.width).
2. return `divider_recalc_rate(&self.hw, parent_rate, val, self.table, self.flags, self.width)`.

`ClkDivider::bestdiv(hw, parent, rate, best_parent_rate, table, width, flags) -> i32`:
1. if rate == 0: rate = 1.
2. maxdiv = get_maxdiv(table, width, flags).
3. if !(clk_hw_get_flags(hw) & CLK_SET_RATE_PARENT):
   - parent_rate = *best_parent_rate.
   - bestdiv = `_div_round(table, parent_rate, rate, flags)`.
   - bestdiv = clamp(bestdiv, 1, maxdiv).
   - return bestdiv.
4. maxdiv = min(ULONG_MAX / rate, maxdiv).
5. let mut best = 0; let mut bestdiv = 0; let parent_rate_saved = *best_parent_rate.
6. for i = next_div(table, 0, flags); i ≤ maxdiv; i = next_div(table, i, flags):
   - if rate * i == parent_rate_saved: *best_parent_rate = parent_rate_saved; return i.
   - p = clk_hw_round_rate(parent, rate * i).
   - now = DIV_ROUND_UP_ULL(p, i).
   - if is_best_div(rate, now, best, flags): bestdiv = i; best = now; *best_parent_rate = p.
7. if bestdiv == 0:
   - bestdiv = get_maxdiv(table, width, flags).
   - *best_parent_rate = clk_hw_round_rate(parent, 1).
8. return bestdiv.

`divider_get_val(rate, parent_rate, table, width, flags) -> i32`:
1. div = DIV_ROUND_UP_ULL(parent_rate, rate).
2. if !is_valid_div(table, div, flags): return -EINVAL.
3. value = ClkDivider::get_val(table, div, flags, width).
4. return min(value, clk_div_mask(width)).

`ClkDivider::set_rate(&self, rate, parent_rate) -> i32`:
1. value = `divider_get_val(rate, parent_rate, self.table, self.width, self.flags)`.
2. if value < 0: return value.
3. /* Optional lock */
4. let _guard = self.lock.map(|l| l.lock_irqsave());
5. if self.flags & CLK_DIVIDER_HIWORD_MASK:
   - reg = clk_div_mask(self.width) << (self.shift + 16).      // write-enable; no prior read
6. else:
   - reg = self.readl(); reg &= !(clk_div_mask(self.width) << self.shift).
7. reg |= (value as u32) << self.shift.
8. self.writel(reg).
9. /* drop guard */
10. return 0.

`ClkDivider::hw_register(dev, np, name, parent*, flags, reg, shift, width, clk_divider_flags, table, lock) -> Result<*ClkHw, Errno>`:
1. if (clk_divider_flags & CLK_DIVIDER_HIWORD_MASK) ∧ (width + shift > 16): return Err(EINVAL).
2. div = Box::new(ClkDivider {…}).
3. init = ClkInitData {
     name,
     ops: if clk_divider_flags & CLK_DIVIDER_READ_ONLY { &CLK_DIVIDER_RO_OPS } else { &CLK_DIVIDER_OPS },
     flags,
     parent_names | parent_hws | parent_data: per-argument,
     num_parents: 1 if any parent_* provided else 0,
   }.
4. div.hw.init = &init.
5. clk_hw_register(dev, &div.hw).
6. on err: drop(div); return Err.
7. return Ok(&div.hw).
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `roundtrip_get_div_get_val` | INVARIANT | per-flag family: `get_div(get_val(div, flags, width), flags, width) == div` for every valid div ≤ maxdiv. |
| `power_of_two_consistency` | INVARIANT | per-`_POWER_OF_TWO`: every accepted div is `is_power_of_2`. |
| `max_at_zero_boundary` | INVARIANT | per-`_MAX_AT_ZERO`: val == 0 ⟺ div == mask + 1. |
| `even_integers_only_even` | INVARIANT | per-`_EVEN_INTEGERS`: every decoded div is even and ≥ 2. |
| `valid_div_subset_table` | INVARIANT | per-table: `is_valid_div` ⟺ table contains entry with `clkt.div == div`. |
| `hiword_width_fits` | INVARIANT | per-`_HIWORD_MASK`: hw_register rejects width + shift > 16. |
| `bestdiv_in_range` | INVARIANT | per-bestdiv: 1 ≤ bestdiv ≤ maxdiv on success. |
| `bestdiv_overflow_clamp` | INVARIANT | per-bestdiv: when `CLK_SET_RATE_PARENT` set, maxdiv ≤ ULONG_MAX / rate. |
| `set_rate_no_read_under_hiword` | INVARIANT | per-`_HIWORD_MASK`: `set_rate` performs no readl. |
| `recalc_zero_div_no_warn_when_allowed` | INVARIANT | per-`_ALLOW_ZERO`: WARN not raised for div == 0. |
| `ro_ops_no_set_rate` | INVARIANT | per-`clk_divider_ro_ops`: `.set_rate` is None / NULL. |
| `endian_consistency` | INVARIANT | per-`_BIG_ENDIAN`: every reg access via `ioread32be`/`iowrite32be`; otherwise via `readl`/`writel`. |
| `lock_held_during_rmw` | INVARIANT | per-set_rate non-hiword: lock acquired before readl and released after writel. |

### Layer 2: TLA+

`drivers/clk/clk-divider.tla`:
- Per-encoding family (ONE_BASED, POWER_OF_TWO, MAX_AT_ZERO, EVEN_INTEGERS, table, default).
- Per-recalc / determine / set / RO sequence.
- Per-lock provider / hiword race-free atomic.
- Properties:
  - `safety_div_in_range` — per-recalc: 1 ≤ div ≤ maxdiv ∨ (div == 0 ∧ `_ALLOW_ZERO`).
  - `safety_set_rate_value_min_mask` — per-set_rate: encoded value ≤ `clk_div_mask(width)`.
  - `safety_hiword_no_read` — per-`_HIWORD_MASK` ∧ set_rate: no readl precedes writel.
  - `safety_ro_no_set` — per-`_READ_ONLY`: `set_rate` never invoked.
  - `safety_endian_pure` — per-`_BIG_ENDIAN`: all accesses BE; otherwise LE.
  - `liveness_bestdiv_terminates` — per-bestdiv loop: terminates with bestdiv ≥ 1 (else fallback to maxdiv).
  - `liveness_register_unregisters` — per-devm: unbind ⟹ kfree.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ClkDivider::recalc_rate` post: `ret == ceil(parent / div) ∨ (div == 0 ∧ ret == parent)` | `ClkDivider::recalc_rate` |
| `ClkDivider::set_rate` post: register bit-field equals `get_val(get_div(.))` | `ClkDivider::set_rate` |
| `divider_get_val` post: `0 ≤ ret ≤ clk_div_mask(width) ∨ ret == -EINVAL` | `divider_get_val` |
| `divider_recalc_rate` post: returns `parent_rate` iff div == 0 | `divider_recalc_rate` |
| `ClkDivider::bestdiv` post: chosen div is among `_next_div`-iterated candidates | `ClkDivider::bestdiv` |
| `__clk_hw_register_divider` post: `hw.init.ops == &CLK_DIVIDER_RO_OPS` iff `_READ_ONLY` set | `ClkDivider::hw_register` |
| `__devm_clk_hw_register_divider` post: devres callback registered on success | `ClkDivider::devm_hw_register` |
| `clk_divider_ops.set_rate` ≠ None; `clk_divider_ro_ops.set_rate` == None | `CLK_DIVIDER_OPS` / `CLK_DIVIDER_RO_OPS` |

### Layer 4: Verus/Creusot functional

`Per-divider hardware contract`: for every `(parent_rate, div)` in the supported family, `recalc_rate(set_rate(rate)) ≈ rate` within rounding policy (`_ROUND_CLOSEST` or round-down-not-exceeding). Encoding/decoding round-trip equivalence over `_ONE_BASED`, `_POWER_OF_TWO`, `_MAX_AT_ZERO`, `_EVEN_INTEGERS`, table, default; per-Documentation/driver-api/clk.rst and per-vendor SoC clock-tree behavioural compatibility.

## Hardening

(Inherits row-1 features from `drivers/clk/00-overview.md` § Hardening.)

Divider reinforcement:

- **Per-zero-divisor guarded** — defense against per-divide-by-zero kernel crash; `_ALLOW_ZERO` opts-in to "div == 0 means parent_rate".
- **Per-`_HIWORD_MASK` width+shift ≤ 16 enforced at register** — defense against per-out-of-LOWORD overflow corrupting siblings.
- **Per-`_POWER_OF_TWO` validity check** — defense against per-non-PoT divisor accepted on a PoT-only hardware (rejects with -EINVAL).
- **Per-table-validity check** — defense against per-out-of-table divisor; `_is_valid_div` gate prevents bogus register encoding.
- **Per-bestdiv overflow clamp `min(ULONG_MAX / rate, maxdiv)`** — defense against per-`rate * i` overflow.
- **Per-provider-spinlock around RMW** — defense against per-concurrent-sibling-clock register tear.
- **Per-`_HIWORD_MASK` lockless atomic write** — defense against per-RMW window for SoC families that hardware-arbitrate.
- **Per-`_READ_ONLY` no `.set_rate` op** — defense against per-misregistered RO-clock write.
- **Per-`_BIG_ENDIAN` endian-pure** — defense against per-byte-swap silent corruption.
- **Per-devres cleanup on unbind** — defense against per-leak when device probe fails after divider register.
- **Per-WARN on zero-divisor when `!_ALLOW_ZERO`** — defense against per-silent-malformed-HW state propagating up the tree.
- **Per-`min(value, clk_div_mask(width))` in `divider_get_val`** — defense against per-bit-field overflow into adjacent fields.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/clk/clk.c` core framework (covered in `clk-core.md` Tier-3).
- `drivers/clk/clk-gate.c` gate primitive (covered separately).
- `drivers/clk/clk-mux.c` multiplexer primitive (covered in `clk-mux.md` Tier-3).
- `drivers/clk/clk-fractional-divider.c` (separate Tier-3 if expanded).
- `drivers/clk/clk-composite.c` composition wrapper (separate Tier-3 if expanded).
- Vendor SoC clock-tree topology (per-vendor Tier-3 if expanded).
- Implementation code.
