---
title: "Tier-3: drivers/clk/clk-mux.c — Generic clock multiplexer"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-11
updated: 2026-05-11
---


## Design Specification

### Summary

The generic **clock multiplexer** is a reusable hardware-block driver: it models an MMIO bit-field that selects one of N parent clocks. Per-block: `struct clk_mux` (reg + shift + mask + flags + optional `u32 *table` + optional `spinlock_t`). Per-rate: rate is **not** adjustable directly; `determine_rate` walks parents via `clk_mux_determine_rate_flags`; `get_parent` reads the bit-field and maps register-value ⟶ parent-index; `set_parent` maps index ⟶ register-value and writes (optionally locked, optionally hiword-mask atomic-RMW). Per-mapping family: optional explicit `table[index] = register_value`, or default-encoding controlled by `CLK_MUX_INDEX_BIT` (`val = 1 << index` — one-hot) and `CLK_MUX_INDEX_ONE` (`val = index + 1` — one-based offset). Per-`CLK_MUX_HIWORD_MASK`: same Rockchip-style atomic-RMW as the divider (top-16-bits write-enable). Per-`CLK_MUX_READ_ONLY`: install ops without `.set_parent`; `clk_set_parent` rejected. Per-`CLK_MUX_BIG_ENDIAN`: BE register access. Per-`CLK_MUX_ROUND_CLOSEST`: rounding policy passed through `clk_mux_determine_rate_flags`. Critical for: SoC clock-tree source selection, ABI-compatible re-implementation of vendor clock drivers built atop the generic mux primitive.

This Tier-3 covers `drivers/clk/clk-mux.c` (~285 lines).

### Acceptance Criteria

- [ ] AC-1: `clk_mux_get_parent` reads register, masks, decodes via `val_to_index`.
- [ ] AC-2: `clk_mux_set_parent` encodes via `index_to_val`, masks, writes.
- [ ] AC-3: `CLK_MUX_INDEX_BIT` round-trip: `val_to_index(index_to_val(i)) == i` for `i < num_parents`.
- [ ] AC-4: `CLK_MUX_INDEX_ONE` round-trip: `val_to_index(index_to_val(i)) == i` for `i < num_parents`.
- [ ] AC-5: Table-based round-trip: `val_to_index(index_to_val(i)) == i` for any valid `i`.
- [ ] AC-6: `val_to_index` with table and val not in table: returns -EINVAL.
- [ ] AC-7: `val_to_index` without table, decoded ≥ num_parents: returns -EINVAL.
- [ ] AC-8: `CLK_MUX_HIWORD_MASK`: `set_parent` writes `(mask << (shift+16)) | (val << shift)` without prior read.
- [ ] AC-9: `CLK_MUX_HIWORD_MASK` + width+shift > 16: registration fails with -EINVAL.
- [ ] AC-10: `CLK_MUX_READ_ONLY`: ops table is `clk_mux_ro_ops` (no `.set_parent`, no `.determine_rate`).
- [ ] AC-11: `CLK_MUX_BIG_ENDIAN`: reads via `ioread32be`, writes via `iowrite32be`.
- [ ] AC-12: Provider lock held during read-modify-write when `mux.lock != NULL`.
- [ ] AC-13: `clk_mux_determine_rate` delegates to `clk_mux_determine_rate_flags(hw, req, mux.flags)`.
- [ ] AC-14: `__devm_clk_hw_register_mux` auto-unregisters on device unbind.
- [ ] AC-15: `__clk_hw_register_mux(dev=NULL, np=valid)` uses `of_clk_hw_register`.

### Architecture

```
struct ClkMux {
  hw: ClkHw,
  reg: *mut MmioReg,
  table: Option<&'static [u32]>,     // optional explicit register-value table
  mask: u32,                          // raw mask, NOT shifted
  shift: u8,
  flags: u8,                          // CLK_MUX_*
  lock: Option<*Spinlock>,
}

const CLK_MUX_INDEX_BIT:     u8 = 1 << 0;
const CLK_MUX_INDEX_ONE:     u8 = 1 << 1;
const CLK_MUX_HIWORD_MASK:   u8 = 1 << 2;
const CLK_MUX_READ_ONLY:     u8 = 1 << 3;
const CLK_MUX_ROUND_CLOSEST: u8 = 1 << 4;
const CLK_MUX_BIG_ENDIAN:    u8 = 1 << 5;
```

`ClkMux::readl(&self) -> u32`:
1. if `self.flags & CLK_MUX_BIG_ENDIAN`: return `ioread32be(self.reg)`.
2. else: return `readl(self.reg)`.

`ClkMux::writel(&self, val: u32)`:
1. if `self.flags & CLK_MUX_BIG_ENDIAN`: `iowrite32be(val, self.reg)`.
2. else: `writel(val, self.reg)`.

`clk_mux_val_to_index(hw, table, flags, val) -> i32`:
1. num_parents = `clk_hw_get_num_parents(hw)`.
2. if table.is_some():
   - for i in 0..num_parents: if table[i] == val: return i as i32.
   - return -EINVAL.
3. if val != 0 ∧ (flags & CLK_MUX_INDEX_BIT): val = ffs(val) - 1.
4. if val != 0 ∧ (flags & CLK_MUX_INDEX_ONE): val -= 1.
5. if val ≥ num_parents: return -EINVAL.
6. return val as i32.

`clk_mux_index_to_val(table, flags, index) -> u32`:
1. let mut val = index as u32.
2. if table.is_some(): val = table[index].
3. else:
   - if flags & CLK_MUX_INDEX_BIT: val = 1 << index.
   - if flags & CLK_MUX_INDEX_ONE: val += 1.
4. return val.

`ClkMux::get_parent(hw) -> u8`:
1. let mux = ClkMux::from_hw(hw).
2. let val = (mux.readl() >> mux.shift) & mux.mask.
3. return `clk_mux_val_to_index(hw, mux.table, mux.flags, val)` as u8.

`ClkMux::set_parent(hw, index: u8) -> i32`:
1. let mux = ClkMux::from_hw(hw).
2. let val = `clk_mux_index_to_val(mux.table, mux.flags, index)`.
3. let _guard = mux.lock.map(|l| l.lock_irqsave());
4. let reg = if mux.flags & CLK_MUX_HIWORD_MASK:
     mux.mask << (mux.shift + 16)              // write-enable mask only
   else:
     mux.readl() & !(mux.mask << mux.shift);   // RMW preserve siblings
5. let reg = reg | (val << mux.shift).
6. mux.writel(reg).
7. /* drop guard */
8. return 0.

`ClkMux::determine_rate(hw, req) -> i32`:
1. let mux = ClkMux::from_hw(hw).
2. return `clk_mux_determine_rate_flags(hw, req, mux.flags)`.

`ClkMux::hw_register(dev, np, name, num_parents, parent_names | parent_hws | parent_data, flags, reg, shift, mask, clk_mux_flags, table, lock) -> Result<*ClkHw, Errno>`:
1. if (clk_mux_flags & CLK_MUX_HIWORD_MASK):
   - width = fls(mask) - ffs(mask) + 1.
   - if width + shift > 16: pr_err; return Err(EINVAL).
2. mux = Box::new(ClkMux {…}).
3. init = ClkInitData {
     name,
     ops: if clk_mux_flags & CLK_MUX_READ_ONLY { &CLK_MUX_RO_OPS } else { &CLK_MUX_OPS },
     flags,
     parent_names, parent_hws, parent_data, num_parents,
   }.
4. mux.hw.init = &init.
5. if dev.is_some() ∨ np.is_none(): clk_hw_register(dev, &mux.hw).
   elif np.is_some(): of_clk_hw_register(np, &mux.hw).
6. on err: drop(mux); return Err.
7. return Ok(&mux.hw).
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `roundtrip_val_to_index_to_val_default` | INVARIANT | per-no-table, no flags: `val_to_index(index_to_val(i)) == i` for i < num_parents. |
| `roundtrip_index_bit` | INVARIANT | per-`CLK_MUX_INDEX_BIT`: `val_to_index(1 << i) == i` for i < num_parents. |
| `roundtrip_index_one` | INVARIANT | per-`CLK_MUX_INDEX_ONE`: `val_to_index(i + 1) == i` for i < num_parents. |
| `roundtrip_table` | INVARIANT | per-table: `val_to_index(table, table[i]) == i` for i < num_parents. |
| `val_to_index_rejects_unknown_table` | INVARIANT | per-table: val not in table[0..num_parents] ⟹ -EINVAL. |
| `val_to_index_rejects_overflow_default` | INVARIANT | per-no-table: decoded ≥ num_parents ⟹ -EINVAL. |
| `hiword_width_fits` | INVARIANT | per-`CLK_MUX_HIWORD_MASK`: hw_register rejects width + shift > 16. |
| `set_parent_no_read_under_hiword` | INVARIANT | per-`CLK_MUX_HIWORD_MASK`: `set_parent` performs no readl. |
| `ro_ops_no_set_parent` | INVARIANT | per-`clk_mux_ro_ops`: `.set_parent` is None / NULL. |
| `endian_consistency` | INVARIANT | per-`CLK_MUX_BIG_ENDIAN`: every access via `ioread32be`/`iowrite32be`; else `readl`/`writel`. |
| `lock_held_during_rmw` | INVARIANT | per-set_parent non-hiword: lock acquired before readl and released after writel. |
| `mask_not_pre_shifted` | INVARIANT | per-`set_parent`: register write uses `(mask << shift)`; `mask` field stores raw mask. |

### Layer 2: TLA+

`drivers/clk/clk-mux.tla`:
- Per-encoding family (default, `INDEX_BIT`, `INDEX_ONE`, table).
- Per-get_parent / set_parent sequence.
- Per-lock provider / hiword race-free atomic.
- Properties:
  - `safety_index_in_range` — per-get_parent: returned index ∈ [0, num_parents) ∨ -EINVAL.
  - `safety_set_parent_value_in_mask` — per-set_parent: encoded register-value-after-shift fits within `mask << shift`.
  - `safety_hiword_no_read` — per-`CLK_MUX_HIWORD_MASK` ∧ set_parent: no readl precedes writel.
  - `safety_ro_no_set` — per-`CLK_MUX_READ_ONLY`: `set_parent` never invoked.
  - `safety_endian_pure` — per-`CLK_MUX_BIG_ENDIAN`: all accesses BE; otherwise LE.
  - `safety_table_precedence` — per-table-set: encoding ignores `INDEX_BIT` / `INDEX_ONE`.
  - `liveness_determine_rate_picks_a_parent` — per-determine_rate: returns 0 with chosen `req.best_parent_hw` populated.
  - `liveness_register_unregisters` — per-devm: unbind ⟹ kfree.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ClkMux::get_parent` post: `0 ≤ ret < num_parents ∨ ret < 0` | `ClkMux::get_parent` |
| `ClkMux::set_parent` post: register bit-field equals `index_to_val(index)` | `ClkMux::set_parent` |
| `clk_mux_val_to_index` post: per-table maps to `i` iff `table[i] == val` | `clk_mux_val_to_index` |
| `clk_mux_index_to_val` post: per-table returns `table[index]` | `clk_mux_index_to_val` |
| `ClkMux::determine_rate` post: `req.best_parent_hw ∈ parents` | `ClkMux::determine_rate` |
| `__clk_hw_register_mux` post: `hw.init.ops == &CLK_MUX_RO_OPS` iff `CLK_MUX_READ_ONLY` set | `ClkMux::hw_register` |
| `__devm_clk_hw_register_mux` post: devres callback registered on success | `ClkMux::devm_hw_register` |
| `clk_mux_ops.set_parent` ≠ None; `clk_mux_ro_ops.set_parent` == None | `CLK_MUX_OPS` / `CLK_MUX_RO_OPS` |

### Layer 4: Verus/Creusot functional

`Per-mux hardware contract`: `get_parent(set_parent(i)) == i` for every `i < num_parents`; consistent with table-or-flag encoding family. `set_parent(i)` preserves all bits outside `mask << shift` (non-hiword path) or atomically updates only the addressed window (hiword path). Encoding/decoding round-trip equivalence over default, `_INDEX_BIT`, `_INDEX_ONE`, table; per-Documentation/driver-api/clk.rst and per-vendor SoC clock-tree behavioural compatibility.

## Hardening

(Inherits row-1 features from `drivers/clk/00-overview.md` § Hardening.)

Mux reinforcement:

- **Per-table-encoded mapping** — defense against per-vendor-irregular bit-pattern (non-monotonic register values per parent).
- **Per-`val_to_index` table-miss → -EINVAL** — defense against per-undefined HW state propagating as parent-index 0.
- **Per-`val_to_index` overflow guard `val ≥ num_parents`** — defense against per-out-of-tree parent reference.
- **Per-`CLK_MUX_HIWORD_MASK` width+shift ≤ 16 enforced at register** — defense against per-out-of-LOWORD overflow corrupting siblings.
- **Per-`CLK_MUX_HIWORD_MASK` lockless atomic write** — defense against per-RMW race on SoC families that hardware-arbitrate.
- **Per-`CLK_MUX_READ_ONLY` no `.set_parent` op** — defense against per-misregistered RO-mux parent-switch.
- **Per-provider-spinlock around RMW** — defense against per-concurrent-sibling-clock register tear.
- **Per-`CLK_MUX_BIG_ENDIAN` endian-pure** — defense against per-byte-swap silent corruption.
- **Per-mask raw (not pre-shifted) discipline** — defense against per-double-shift bit-field encoding bug.
- **Per-`val != 0` guard in `INDEX_BIT` / `INDEX_ONE` decode** — defense against per-zero-bias incorrectly mapping val == 0 to negative parent-index.
- **Per-devres cleanup on unbind** — defense against per-leak when device probe fails after mux register.
- **Per-OF and platform-device dual registration path** — defense against per-binding-mode regression (DT-only vs device-attached).
- **Per-`fls(mask) - ffs(mask) + 1` width derivation** — defense against per-non-contiguous-mask silent acceptance under hiword check.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/clk/clk.c` core framework + `clk_mux_determine_rate_flags` helper (covered in `clk-core.md` Tier-3).
- `drivers/clk/clk-gate.c` gate primitive (covered separately).
- `drivers/clk/clk-divider.c` divider primitive (covered in `clk-divider.md` Tier-3).
- `drivers/clk/clk-composite.c` composition wrapper (separate Tier-3 if expanded).
- Vendor SoC clock-tree topology (per-vendor Tier-3 if expanded).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct clk_mux` | per-mux HW data | `ClkMux` |
| `clk_mux_ops` | per-RW vtable | `CLK_MUX_OPS` |
| `clk_mux_ro_ops` | per-RO vtable | `CLK_MUX_RO_OPS` |
| `to_clk_mux()` | per-`container_of` cast | `ClkMux::from_hw` |
| `clk_mux_readl()` | per-LE/BE register read | `ClkMux::readl` |
| `clk_mux_writel()` | per-LE/BE register write | `ClkMux::writel` |
| `clk_mux_val_to_index()` | per-register-val ⟶ parent-index | `clk_mux_val_to_index` |
| `clk_mux_index_to_val()` | per-parent-index ⟶ register-val | `clk_mux_index_to_val` |
| `clk_mux_get_parent()` | per-`.get_parent` op | `ClkMux::get_parent` |
| `clk_mux_set_parent()` | per-`.set_parent` op | `ClkMux::set_parent` |
| `clk_mux_determine_rate()` | per-`.determine_rate` op | `ClkMux::determine_rate` |
| `clk_mux_determine_rate_flags()` | per-helper (in `clk.c`) | external; called from determine_rate |
| `__clk_hw_register_mux()` | per-`clk_hw` registration core | `ClkMux::hw_register` |
| `__devm_clk_hw_register_mux()` | per-devres-managed registration | `ClkMux::devm_hw_register` |
| `clk_register_mux_table()` | per-legacy `clk*` API | `ClkMux::register_table` |
| `clk_unregister_mux()` / `clk_hw_unregister_mux()` | per-teardown | `ClkMux::unregister` / `hw_unregister` |
| `devm_clk_hw_release_mux()` | per-devres release | `ClkMux::devm_release` |
| `CLK_MUX_INDEX_BIT` | per-encoding `val = 1 << index` | flag bit |
| `CLK_MUX_INDEX_ONE` | per-encoding `val = index + 1` | flag bit |
| `CLK_MUX_HIWORD_MASK` | per-atomic-RMW write style | flag bit |
| `CLK_MUX_READ_ONLY` | per-skip-`set_parent` | flag bit |
| `CLK_MUX_ROUND_CLOSEST` | per-rate-determination policy | flag bit |
| `CLK_MUX_BIG_ENDIAN` | per-BE register access | flag bit |

### compatibility contract

REQ-1: struct clk_mux:
- hw: per-`struct clk_hw` (embedded).
- reg: per-`void __iomem *` MMIO register.
- table: per-optional `const u32 *` — `table[index] = register_value`; NULL ⟹ use flag-driven default encoding.
- mask: per-`u32` raw bit-mask of the parent-select bit-field (NOT pre-shifted; e.g. `0x7` for a 3-bit field).
- shift: per-bit-field LSB position in register (u8).
- flags: per-`CLK_MUX_*` mask (u8).
- lock: per-optional `spinlock_t *` shared with sibling clocks on the same register.

REQ-2: clk_mux_readl(mux) / clk_mux_writel(mux, val):
- if `mux.flags & CLK_MUX_BIG_ENDIAN`: `ioread32be` / `iowrite32be`.
- else: `readl` / `writel`.

REQ-3: clk_mux_val_to_index(hw, table, flags, val) -> int:
- num_parents = `clk_hw_get_num_parents(hw)`.
- if table:
  - for i in 0..num_parents: if `table[i] == val`: return i.
  - return -EINVAL.
- if val ∧ `CLK_MUX_INDEX_BIT`: val = `ffs(val) - 1` (decode one-hot to bit-index).
- if val ∧ `CLK_MUX_INDEX_ONE`: val-- (decode one-based to zero-based).
- if val ≥ num_parents: return -EINVAL.
- return val.

REQ-4: clk_mux_index_to_val(table, flags, index) -> unsigned int:
- val = index.
- if table: val = `table[index]`.
- else:
  - if `CLK_MUX_INDEX_BIT`: val = `1 << index`.
  - if `CLK_MUX_INDEX_ONE`: val++.
- return val.

REQ-5: Per-flag-combination notes:
- `CLK_MUX_INDEX_BIT` and `CLK_MUX_INDEX_ONE` may compose (both transformations applied sequentially in `index_to_val`: shift-to-one-hot then add 1).
- table takes precedence over flag-driven encoding in both directions.
- table must have at least `num_parents` entries.

REQ-6: clk_mux_get_parent(hw) -> u8:
- mux = `to_clk_mux(hw)`.
- val = `clk_mux_readl(mux) >> mux.shift`.
- val &= mux.mask.
- return `clk_mux_val_to_index(hw, mux.table, mux.flags, val)` (cast to u8; -EINVAL surfaced via error code).

REQ-7: clk_mux_set_parent(hw, index) -> int:
- mux = `to_clk_mux(hw)`.
- val = `clk_mux_index_to_val(mux.table, mux.flags, index)`.
- if mux.lock: `spin_lock_irqsave(mux.lock, flags)` else `__acquire(mux.lock)`.
- if `mux.flags & CLK_MUX_HIWORD_MASK`:
  - reg = `mux.mask << (mux.shift + 16)` (write-enable mask only; no prior read).
- else:
  - reg = `clk_mux_readl(mux)`; reg &= `~(mux.mask << mux.shift)`.
- val <<= mux.shift.
- reg |= val.
- `clk_mux_writel(mux, reg)`.
- if mux.lock: `spin_unlock_irqrestore` else `__release(mux.lock)`.
- return 0.

REQ-8: clk_mux_determine_rate(hw, req) -> int:
- mux = `to_clk_mux(hw)`.
- return `clk_mux_determine_rate_flags(hw, req, mux.flags)`.
- /* The helper (defined in `drivers/clk/clk.c`) iterates parents, asks each to round `req.rate`, picks per-`CLK_MUX_ROUND_CLOSEST` policy. */

REQ-9: clk_mux_ops vtable:
- .get_parent = clk_mux_get_parent.
- .set_parent = clk_mux_set_parent.
- .determine_rate = clk_mux_determine_rate.

REQ-10: clk_mux_ro_ops vtable:
- .get_parent = clk_mux_get_parent.
- /* No .set_parent, no .determine_rate — parent fixed in HW from boot. */

REQ-11: __clk_hw_register_mux(dev, np, name, num_parents, parent_names, parent_hws, parent_data, flags, reg, shift, mask, clk_mux_flags, table, lock):
- if `clk_mux_flags & CLK_MUX_HIWORD_MASK`:
  - width = `fls(mask) - ffs(mask) + 1`.
  - if `width + shift > 16`: pr_err("mux value exceeds LOWORD field"); return ERR_PTR(-EINVAL).
- mux = kzalloc(sizeof(*mux), GFP_KERNEL); if !mux: return ERR_PTR(-ENOMEM).
- init.name = name.
- init.ops = `(clk_mux_flags & CLK_MUX_READ_ONLY) ? &clk_mux_ro_ops : &clk_mux_ops`.
- init.flags = flags.
- init.parent_names = parent_names; init.parent_hws = parent_hws; init.parent_data = parent_data.
- init.num_parents = num_parents.
- Populate mux→{reg, shift, mask, flags, lock, table, hw.init}.
- hw = &mux.hw.
- if dev ∨ !np: ret = `clk_hw_register(dev, hw)`.
- elif np: ret = `of_clk_hw_register(np, hw)`.
- if ret: kfree(mux); return ERR_PTR(ret).
- return hw.

REQ-12: __devm_clk_hw_register_mux: same as REQ-11 but devres-tracked via `devres_alloc(devm_clk_hw_release_mux, ...)`; on success `devres_add`; on failure `devres_free`.

REQ-13: clk_register_mux_table(dev, name, parent_names, num_parents, flags, reg, shift, mask, clk_mux_flags, table, lock) -> struct clk *:
- hw = `clk_hw_register_mux_table(dev, name, parent_names, num_parents, flags, reg, shift, mask, clk_mux_flags, table, lock)`.
- if IS_ERR(hw): return ERR_CAST(hw).
- return hw.clk.

REQ-14: clk_unregister_mux(clk):
- hw = `__clk_get_hw(clk)`; if !hw: return.
- mux = `to_clk_mux(hw)`.
- `clk_unregister(clk)`; kfree(mux).

REQ-15: clk_hw_unregister_mux(hw):
- mux = `to_clk_mux(hw)`.
- `clk_hw_unregister(hw)`; kfree(mux).

REQ-16: devm_clk_hw_release_mux(dev, res):
- Unwrap stored `clk_hw **`; call `clk_hw_unregister_mux`.

REQ-17: Locking discipline:
- mux.lock is OPTIONAL.
- If non-NULL: held over read-modify-write (and over the single hiword write).
- If NULL: caller asserts single-writer ownership; sparse `__acquire/__release` balance.

REQ-18: Per-CLK_MUX_HIWORD_MASK semantics:
- Register layout: bits [31:16] = write-enable mask; bits [15:0] = data.
- At register time, `fls(mask) - ffs(mask) + 1 + shift ≤ 16` enforced.
- `set_parent` writes `(mux.mask << (shift + 16)) | (val << shift)` — no prior read; race-free.

REQ-19: Per-table semantics:
- `table[i]` is the raw register-value for parent index `i`.
- Table-based mapping bypasses `CLK_MUX_INDEX_BIT` / `_INDEX_ONE` transforms.
- `val_to_index` does a linear scan; `index_to_val` is direct array lookup.

REQ-20: Per-`CLK_MUX_INDEX_BIT` semantics:
- Hardware uses one-hot encoding (e.g. parent 0 ⟹ val = 1, parent 1 ⟹ val = 2, parent 2 ⟹ val = 4).
- Decode: `val_to_index` applies `ffs(val) - 1` only when `val != 0` (val == 0 with `INDEX_BIT` is treated as direct).
- Encode: `index_to_val` applies `1 << index`.

REQ-21: Per-`CLK_MUX_INDEX_ONE` semantics:
- Hardware index is one-based (e.g. parent 0 ⟹ val = 1, parent 1 ⟹ val = 2, …).
- Decode: subtract 1 (only when `val != 0`).
- Encode: add 1.

REQ-22: Per-mask field, NOT pre-shifted:
- `mask` is the raw `(1 << width) - 1` value of the parent-select field, NOT shifted by `shift`.
- All `<< shift` and `>> shift` happens in `get_parent` / `set_parent`.

REQ-23: Per-rate behavior:
- Mux does not modify rate; it propagates a parent's rate.
- `determine_rate` walks all parents (via `clk_mux_determine_rate_flags`), asks each for `round_rate(req.rate)`, picks per-`CLK_MUX_ROUND_CLOSEST` (closest match) or default (largest-not-exceeding) policy, writes back `req.rate` and `req.best_parent_hw` / `req.best_parent_rate`.

REQ-24: Per-`CLK_MUX_READ_ONLY` boot-time parent: `get_parent` reads the bit-field; `set_parent` is omitted from the vtable (so `clk_set_parent` returns -ENOSYS or equivalent).

