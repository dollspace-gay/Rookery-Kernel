# Tier-3: drivers/regulator/helpers.c — Regulator regmap & linear-range helpers

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/regulator/00-overview.md
upstream-paths:
  - drivers/regulator/helpers.c (~1003 lines)
  - drivers/regulator/internal.h
  - include/linux/regulator/driver.h
  - include/linux/linear_range.h
-->

## Summary

The **regulator helpers** module supplies the canonical implementations of the most common `struct regulator_ops` callbacks so that the ~50+ PMIC and SoC-internal regulator drivers do not each re-implement register-poking, voltage-table search, and linear-range arithmetic. Two orthogonal axes are provided. Per-**regmap helpers**: drivers that store enable/disable, voltage-selector, bypass, soft-start, pull-down, active-discharge, current-limit, and ramp-delay state in regmap-addressable registers populate `enable_reg/enable_mask/enable_val`, `vsel_reg/vsel_mask`, `vsel_range_reg/vsel_range_mask`, `bypass_reg/bypass_mask/bypass_val_on/off`, etc. in their `regulator_desc` and reference the helper as the corresponding op (e.g. `.enable = regulator_enable_regmap`). Per-**voltage-list helpers**: drivers expose either a linear (min_uV + uV_step * sel), a tabular (`volt_table[sel]`), or a multi-`linear_range`d (`struct linear_range[]`) voltage map; helpers `regulator_list_voltage_linear`, `_table`, `_linear_range`, `_pickable_linear_range` translate selector ↔ uV. Per-**pickable ranges**: the PMIC stores the range-index in a separate `vsel_range_reg/_mask` field; the helper walks `linear_range_selectors_bitfield[]` to map between selector-tuples and the global selector. Per-**map_voltage**: `regulator_map_voltage_iterate` brute-forces via `list_voltage`; `_ascend` early-exits when list is monotonic; `_linear` does `DIV_ROUND_UP`; `_linear_range` walks ranges. Per-current-limit and ramp-delay regmap helpers complete the inventory. Critical for: PMIC driver brevity, uniform fault behaviour across vendors, regulator-core test coverage.

This Tier-3 covers `drivers/regulator/helpers.c` (~1003 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `regulator_is_enabled_regmap()` | per-regmap is_enabled | `Helpers::is_enabled_regmap` |
| `regulator_enable_regmap()` | per-regmap enable | `Helpers::enable_regmap` |
| `regulator_disable_regmap()` | per-regmap disable | `Helpers::disable_regmap` |
| `regulator_get_voltage_sel_regmap()` | per-regmap get_vsel | `Helpers::get_voltage_sel_regmap` |
| `regulator_set_voltage_sel_regmap()` | per-regmap set_vsel | `Helpers::set_voltage_sel_regmap` |
| `regulator_get_voltage_sel_pickable_regmap()` | per-pickable get_vsel | `Helpers::get_voltage_sel_pickable_regmap` |
| `regulator_set_voltage_sel_pickable_regmap()` | per-pickable set_vsel | `Helpers::set_voltage_sel_pickable_regmap` |
| `regulator_range_selector_to_index()` | per-pickable range lookup | `Helpers::range_selector_to_index` |
| `write_separate_vsel_and_range()` | per-pickable split-reg write | `Helpers::write_separate_vsel_and_range` |
| `regulator_map_voltage_iterate()` | per-list iterate map | `Helpers::map_voltage_iterate` |
| `regulator_map_voltage_ascend()` | per-ascendant map | `Helpers::map_voltage_ascend` |
| `regulator_map_voltage_linear()` | per-linear map | `Helpers::map_voltage_linear` |
| `regulator_map_voltage_linear_range()` | per-linear-range map | `Helpers::map_voltage_linear_range` |
| `regulator_map_voltage_pickable_linear_range()` | per-pickable map | `Helpers::map_voltage_pickable_linear_range` |
| `regulator_desc_list_voltage_linear()` | per-desc list (linear) | `Helpers::desc_list_voltage_linear` |
| `regulator_list_voltage_linear()` | per-rdev list (linear) | `Helpers::list_voltage_linear` |
| `regulator_desc_list_voltage_linear_range()` | per-desc list (ranges) | `Helpers::desc_list_voltage_linear_range` |
| `regulator_list_voltage_linear_range()` | per-rdev list (ranges) | `Helpers::list_voltage_linear_range` |
| `regulator_list_voltage_pickable_linear_range()` | per-rdev pickable list | `Helpers::list_voltage_pickable_linear_range` |
| `regulator_list_voltage_table()` | per-table list | `Helpers::list_voltage_table` |
| `regulator_set_bypass_regmap()` | per-regmap bypass set | `Helpers::set_bypass_regmap` |
| `regulator_get_bypass_regmap()` | per-regmap bypass get | `Helpers::get_bypass_regmap` |
| `regulator_set_soft_start_regmap()` | per-regmap soft-start | `Helpers::set_soft_start_regmap` |
| `regulator_set_pull_down_regmap()` | per-regmap pull-down | `Helpers::set_pull_down_regmap` |
| `regulator_set_active_discharge_regmap()` | per-regmap discharge | `Helpers::set_active_discharge_regmap` |
| `regulator_set_current_limit_regmap()` | per-regmap set_curr_lim | `Helpers::set_current_limit_regmap` |
| `regulator_get_current_limit_regmap()` | per-regmap get_curr_lim | `Helpers::get_current_limit_regmap` |
| `regulator_set_ramp_delay_regmap()` | per-regmap ramp-delay | `Helpers::set_ramp_delay_regmap` |
| `regulator_find_closest_bigger()` | per-table closest-bigger | `Helpers::find_closest_bigger` |
| `regulator_bulk_set_supply_names()` | per-bulk init helper | `Helpers::bulk_set_supply_names` |
| `regulator_is_equal()` | per-identity check | `Helpers::is_equal` |

## Compatibility contract

REQ-1: regulator_is_enabled_regmap(rdev):
- regmap_read(rdev.regmap, desc.enable_reg, &val); if err: return err.
- val &= desc.enable_mask.
- if desc.enable_is_inverted:
  - if desc.enable_val: return val != desc.enable_val.
  - else: return val == 0.
- else:
  - if desc.enable_val: return val == desc.enable_val.
  - else: return val != 0.

REQ-2: regulator_enable_regmap(rdev):
- if desc.enable_is_inverted: val = desc.disable_val.
- else: val = desc.enable_val; if !val: val = desc.enable_mask.
- return regmap_update_bits(regmap, desc.enable_reg, desc.enable_mask, val).

REQ-3: regulator_disable_regmap(rdev):
- if desc.enable_is_inverted: val = desc.enable_val; if !val: val = desc.enable_mask.
- else: val = desc.disable_val.
- return regmap_update_bits(regmap, desc.enable_reg, desc.enable_mask, val).

REQ-4: regulator_get_voltage_sel_regmap(rdev):
- regmap_read(regmap, desc.vsel_reg, &val); if err: return err.
- val &= desc.vsel_mask.
- val >>= ffs(desc.vsel_mask) - 1.
- return val.

REQ-5: regulator_set_voltage_sel_regmap(rdev, sel):
- sel <<= ffs(desc.vsel_mask) - 1.
- ret = regmap_update_bits(regmap, desc.vsel_reg, desc.vsel_mask, sel); if err: return err.
- if desc.apply_bit:
  - ret = regmap_update_bits(regmap, desc.apply_reg, desc.apply_bit, desc.apply_bit).
- return ret.

REQ-6: regulator_range_selector_to_index(rdev, rval) [static]:
- if !desc.linear_range_selectors_bitfield: return -EINVAL.
- rval &= desc.vsel_range_mask.
- rval >>= ffs(desc.vsel_range_mask) - 1.
- for i in 0..desc.n_linear_ranges:
  - if desc.linear_range_selectors_bitfield[i] == rval: return i.
- return -EINVAL.

REQ-7: regulator_get_voltage_sel_pickable_regmap(rdev):
- r = desc.linear_ranges; if !r: return -EINVAL.
- regmap_read(regmap, desc.vsel_reg, &val); if err: return err.
- regmap_read(regmap, desc.vsel_range_reg, &r_val); if err: return err.
- val &= desc.vsel_mask; val >>= ffs(desc.vsel_mask) - 1.
- range = regulator_range_selector_to_index(rdev, r_val); if range < 0: return -EINVAL.
- voltages = linear_range_values_in_range_array(r, range).
- return val + voltages.

REQ-8: write_separate_vsel_and_range(rdev, sel, range) [static]:
- regmap_update_bits_base(regmap, desc.vsel_range_reg, desc.vsel_range_mask, range, &range_updated, false, false).
- if desc.range_applied_by_vsel && range_updated:
  - return regmap_write_bits(regmap, desc.vsel_reg, desc.vsel_mask, sel) (force-write even if unchanged).
- return regmap_update_bits(regmap, desc.vsel_reg, desc.vsel_mask, sel).

REQ-9: regulator_set_voltage_sel_pickable_regmap(rdev, sel):
- for i in 0..desc.n_linear_ranges:
  - r = &desc.linear_ranges[i].
  - voltages_in_range = linear_range_values_in_range(r).
  - if sel < voltages_in_range: break.
  - sel -= voltages_in_range.
- if i == desc.n_linear_ranges: return -EINVAL.
- sel <<= ffs(desc.vsel_mask) - 1.
- sel += desc.linear_ranges[i].min_sel.
- range = desc.linear_range_selectors_bitfield[i].
- range <<= ffs(desc.vsel_range_mask) - 1.
- if desc.vsel_reg == desc.vsel_range_reg:
  - ret = regmap_update_bits(regmap, desc.vsel_reg, desc.vsel_range_mask | desc.vsel_mask, sel | range).
- else:
  - ret = write_separate_vsel_and_range(rdev, sel, range).
- if err: return err.
- if desc.apply_bit:
  - ret = regmap_update_bits(regmap, desc.apply_reg, desc.apply_bit, desc.apply_bit).
- return ret.

REQ-10: regulator_map_voltage_iterate(rdev, min_uV, max_uV):
- best_val = INT_MAX; selector = 0.
- for i in 0..desc.n_voltages:
  - ret = desc.ops.list_voltage(rdev, i); if ret < 0: continue.
  - if ret < best_val ∧ ret >= min_uV ∧ ret <= max_uV: best_val = ret; selector = i.
- return (best_val != INT_MAX) ? selector : -EINVAL.

REQ-11: regulator_map_voltage_ascend(rdev, min_uV, max_uV):
- for i in 0..desc.n_voltages:
  - ret = desc.ops.list_voltage(rdev, i); if ret < 0: continue.
  - if ret > max_uV: break (monotonic).
  - if ret >= min_uV ∧ ret <= max_uV: return i.
- return -EINVAL.

REQ-12: regulator_map_voltage_linear(rdev, min_uV, max_uV):
- /* Allow uV_step == 0 for genuinely-fixed regulator */
- if desc.n_voltages == 1 ∧ desc.uV_step == 0:
  - return (min_uV <= desc.min_uV ∧ desc.min_uV <= max_uV) ? 0 : -EINVAL.
- if !desc.uV_step: BUG_ON; return -EINVAL.
- if min_uV < desc.min_uV: min_uV = desc.min_uV.
- ret = DIV_ROUND_UP(min_uV - desc.min_uV, desc.uV_step).
- if ret < 0: return ret.
- ret += desc.linear_min_sel.
- voltage = desc.ops.list_voltage(rdev, ret).
- if voltage < min_uV ∨ voltage > max_uV: return -EINVAL.
- return ret.

REQ-13: regulator_map_voltage_linear_range(rdev, min_uV, max_uV):
- if !desc.n_linear_ranges: BUG_ON; return -EINVAL.
- for i in 0..desc.n_linear_ranges:
  - range = &desc.linear_ranges[i].
  - linear_range_get_selector_high(range, min_uV, &sel, &found); if err: continue.
  - ret = sel.
  - voltage = desc.ops.list_voltage(rdev, sel).
  - if voltage >= min_uV ∧ voltage <= max_uV: break.
- if i == desc.n_linear_ranges: return -EINVAL.
- return ret.

REQ-14: regulator_map_voltage_pickable_linear_range(rdev, min_uV, max_uV):
- if !desc.n_linear_ranges: BUG_ON; return -EINVAL.
- selector = 0; ret = -EINVAL.
- for i in 0..desc.n_linear_ranges:
  - range = &desc.linear_ranges[i].
  - linear_max_uV = linear_range_get_max_value(range).
  - if !(min_uV <= linear_max_uV ∧ max_uV >= range.min):
    - selector += linear_range_values_in_range(range); continue.
  - linear_range_get_selector_high(range, min_uV, &sel, &found); if err: selector += linear_range_values_in_range(range); continue.
  - ret = selector + sel - range.min_sel.
  - voltage = desc.ops.list_voltage(rdev, ret).
  - /* overlapping ranges: keep searching until bounds match */
  - if voltage < min_uV ∨ voltage > max_uV: selector += linear_range_values_in_range(range).
  - else: break.
- if i == desc.n_linear_ranges: return -EINVAL.
- return ret.

REQ-15: regulator_desc_list_voltage_linear(desc, selector):
- if selector >= desc.n_voltages: return -EINVAL.
- if selector < desc.linear_min_sel: return 0.
- selector -= desc.linear_min_sel.
- return desc.min_uV + desc.uV_step * selector.

REQ-16: regulator_list_voltage_linear(rdev, selector):
- return regulator_desc_list_voltage_linear(rdev.desc, selector).

REQ-17: regulator_list_voltage_pickable_linear_range(rdev, selector):
- if !desc.n_linear_ranges: BUG_ON; return -EINVAL.
- all_sels = 0.
- for i in 0..desc.n_linear_ranges:
  - range = &desc.linear_ranges[i].
  - sel_indexes = linear_range_values_in_range(range) - 1.
  - if all_sels + sel_indexes >= selector:
    - selector -= all_sels.
    - return range.min + range.step * selector.
  - all_sels += sel_indexes + 1.
- return -EINVAL.

REQ-18: regulator_desc_list_voltage_linear_range(desc, selector):
- BUG_ON(!desc.n_linear_ranges).
- linear_range_get_value_array(desc.linear_ranges, desc.n_linear_ranges, selector, &val); if err: return err.
- return val.

REQ-19: regulator_list_voltage_linear_range(rdev, selector):
- return regulator_desc_list_voltage_linear_range(rdev.desc, selector).

REQ-20: regulator_list_voltage_table(rdev, selector):
- if !desc.volt_table: BUG_ON; return -EINVAL.
- if selector >= desc.n_voltages: return -EINVAL.
- if selector < desc.linear_min_sel: return 0.
- return desc.volt_table[selector].

REQ-21: regulator_set_bypass_regmap(rdev, enable):
- if enable: val = desc.bypass_val_on; if !val: val = desc.bypass_mask.
- else: val = desc.bypass_val_off.
- return regmap_update_bits(regmap, desc.bypass_reg, desc.bypass_mask, val).

REQ-22: regulator_get_bypass_regmap(rdev, *enable):
- regmap_read(regmap, desc.bypass_reg, &val); if err: return err.
- val_on = desc.bypass_val_on; if !val_on: val_on = desc.bypass_mask.
- *enable = (val & desc.bypass_mask) == val_on.
- return 0.

REQ-23: regulator_set_soft_start_regmap(rdev):
- val = desc.soft_start_val_on; if !val: val = desc.soft_start_mask.
- return regmap_update_bits(regmap, desc.soft_start_reg, desc.soft_start_mask, val).

REQ-24: regulator_set_pull_down_regmap(rdev):
- val = desc.pull_down_val_on; if !val: val = desc.pull_down_mask.
- return regmap_update_bits(regmap, desc.pull_down_reg, desc.pull_down_mask, val).

REQ-25: regulator_set_active_discharge_regmap(rdev, enable):
- val = enable ? desc.active_discharge_on : desc.active_discharge_off.
- return regmap_update_bits(regmap, desc.active_discharge_reg, desc.active_discharge_mask, val).

REQ-26: regulator_set_current_limit_regmap(rdev, min_uA, max_uA):
- n_currents = desc.n_current_limits; if n_currents == 0: return -EINVAL.
- if desc.curr_table:
  - ascend = curr_table[n-1] > curr_table[0].
  - if ascend: iterate i from n-1 down to 0; pick first within [min_uA, max_uA]; sel = i; break.
  - else: iterate i from 0 up to n-1; pick first within [min_uA, max_uA]; sel = i; break.
- if sel < 0: return -EINVAL.
- sel <<= ffs(desc.csel_mask) - 1.
- return regmap_update_bits(regmap, desc.csel_reg, desc.csel_mask, sel).

REQ-27: regulator_get_current_limit_regmap(rdev):
- regmap_read(regmap, desc.csel_reg, &val); if err: return err.
- val &= desc.csel_mask; val >>= ffs(desc.csel_mask) - 1.
- if desc.curr_table:
  - if val >= desc.n_current_limits: return -EINVAL.
  - return desc.curr_table[val].
- return -EINVAL.

REQ-28: regulator_find_closest_bigger(target, table, num_sel, *sel):
- max = table[0]; maxsel = 0; found = false.
- for s in 0..num_sel:
  - if table[s] > max: max = table[s]; maxsel = s.
  - if table[s] >= target:
    - if !found ∨ table[s] - target < tmp - target: tmp = table[s]; *sel = s; found = true; if tmp == target: break.
- if !found: *sel = maxsel; return -EINVAL.
- return 0.

REQ-29: regulator_set_ramp_delay_regmap(rdev, ramp_delay):
- if WARN_ON(!desc.n_ramp_values ∨ !desc.ramp_delay_table): return -EINVAL.
- ret = regulator_find_closest_bigger(ramp_delay, desc.ramp_delay_table, desc.n_ramp_values, &sel).
- if ret: dev_warn("Can't set ramp-delay %u, setting %u").
- sel <<= ffs(desc.ramp_mask) - 1.
- return regmap_update_bits(regmap, desc.ramp_reg, desc.ramp_mask, sel).

REQ-30: regulator_bulk_set_supply_names(consumers, supply_names, num_supplies):
- for i in 0..num_supplies: consumers[i].supply = supply_names[i].

REQ-31: regulator_is_equal(reg1, reg2):
- return reg1.rdev == reg2.rdev.

REQ-32: All EXPORT_SYMBOL_GPL — these helpers are part of the in-tree regulator-driver kABI consumed by ~50+ PMIC drivers; per-symbol export is GPL-only by license discipline.

REQ-33: enable-value semantics — per-`enable_val == 0` means "any non-zero in mask = enabled" (legacy compat); per-`enable_val != 0` means "exact-match required". Per-`enable_is_inverted` flips both senses. The same dichotomy applies to bypass_val_on / soft_start_val_on / pull_down_val_on.

REQ-34: Selector shift normalization — per-vsel/csel/ramp mask: the helper canonically right-shifts by `ffs(mask) - 1` after read and left-shifts by the same amount before write, so callers see selector values starting at 0 regardless of the mask's bit position.

## Acceptance Criteria

- [ ] AC-1: enable_reg=0x10, enable_mask=0x01, enable_val=0: regulator_enable_regmap writes 0x01 to bit-0 of 0x10.
- [ ] AC-2: enable_is_inverted=true, disable_val=0: regulator_enable_regmap writes 0 (the inverted "on" value).
- [ ] AC-3: regulator_is_enabled_regmap with enable_val=0x01, mask=0x03, register value 0x01: returns true.
- [ ] AC-4: regulator_is_enabled_regmap with enable_is_inverted=true, mask=0x01, register value 0x01: returns false.
- [ ] AC-5: regulator_get_voltage_sel_regmap with vsel_mask=0x70: register 0x40 → selector 4.
- [ ] AC-6: regulator_set_voltage_sel_regmap with vsel_mask=0x70, sel=2: writes 0x20 to vsel_reg.
- [ ] AC-7: apply_bit set: set_voltage_sel_regmap also touches apply_reg|apply_bit afterward.
- [ ] AC-8: regulator_list_voltage_linear: min_uV=600000, uV_step=12500, sel=4, linear_min_sel=0: returns 650000.
- [ ] AC-9: regulator_list_voltage_linear: selector >= n_voltages: returns -EINVAL.
- [ ] AC-10: regulator_list_voltage_linear: selector < linear_min_sel: returns 0.
- [ ] AC-11: regulator_list_voltage_table: returns volt_table[selector].
- [ ] AC-12: regulator_list_voltage_table: selector >= n_voltages: returns -EINVAL.
- [ ] AC-13: regulator_map_voltage_linear: min_uV=820000, max_uV=850000, min_uV/uV_step=12500: returns ceiling selector mapping into range; verifies via list_voltage that result is in [min_uV, max_uV].
- [ ] AC-14: regulator_map_voltage_linear with uV_step=0, n_voltages=1, min_uV in range: returns 0.
- [ ] AC-15: regulator_map_voltage_iterate vs _ascend: same input range → identical selector (when monotonic).
- [ ] AC-16: pickable_regmap with two ranges (sel-0..63 + sel-64..127), correct decoding: get→correct, set→writes both vsel_reg and vsel_range_reg.
- [ ] AC-17: pickable get with unknown range value: returns -EINVAL.
- [ ] AC-18: set_bypass_regmap(true)+get_bypass_regmap reads back true.
- [ ] AC-19: set_soft_start_regmap writes soft_start_val_on (or mask if val_on==0).
- [ ] AC-20: set_current_limit_regmap with ascending curr_table: picks largest entry within [min_uA, max_uA].
- [ ] AC-21: set_current_limit_regmap with no curr_table: returns -EINVAL.
- [ ] AC-22: set_ramp_delay_regmap with empty ramp_delay_table: WARN + returns -EINVAL.
- [ ] AC-23: find_closest_bigger: target=120, table=[100,150,200]: returns 1 (sel=1, value=150).
- [ ] AC-24: find_closest_bigger: target larger than any table entry: returns -EINVAL with *sel = index of max.
- [ ] AC-25: regulator_is_equal with two regulators sharing rdev: returns true.

## Architecture

```
struct RegulatorDesc {                       // (subset relevant to helpers)
  ops: *RegulatorOps,
  n_voltages: u32,
  n_current_limits: u32,
  min_uV: i32,
  uV_step: u32,
  linear_min_sel: u32,
  volt_table: Option<&'static [u32]>,
  linear_ranges: Option<&'static [LinearRange]>,
  linear_range_selectors_bitfield: Option<&'static [u32]>,
  n_linear_ranges: u32,
  enable_reg: u32,
  enable_mask: u32,
  enable_val: u32,
  disable_val: u32,
  enable_is_inverted: bool,
  vsel_reg: u32,
  vsel_mask: u32,
  vsel_range_reg: u32,
  vsel_range_mask: u32,
  range_applied_by_vsel: bool,
  apply_reg: u32,
  apply_bit: u32,
  bypass_reg: u32,
  bypass_mask: u32,
  bypass_val_on: u32,
  bypass_val_off: u32,
  soft_start_reg: u32,
  soft_start_mask: u32,
  soft_start_val_on: u32,
  pull_down_reg: u32,
  pull_down_mask: u32,
  pull_down_val_on: u32,
  active_discharge_reg: u32,
  active_discharge_mask: u32,
  active_discharge_on: u32,
  active_discharge_off: u32,
  csel_reg: u32,
  csel_mask: u32,
  curr_table: Option<&'static [u32]>,
  ramp_reg: u32,
  ramp_mask: u32,
  ramp_delay_table: Option<&'static [u32]>,
  n_ramp_values: u32,
}
```

`Helpers::is_enabled_regmap(rdev) -> i32`:
1. regmap_read(rdev.regmap, desc.enable_reg, &mut val)?.
2. val &= desc.enable_mask.
3. if desc.enable_is_inverted:
   - if desc.enable_val != 0: return (val != desc.enable_val) as i32.
   - else: return (val == 0) as i32.
4. else:
   - if desc.enable_val != 0: return (val == desc.enable_val) as i32.
   - else: return (val != 0) as i32.

`Helpers::enable_regmap(rdev) -> Result<(), Errno>`:
1. let val = if desc.enable_is_inverted {
       desc.disable_val
   } else if desc.enable_val != 0 {
       desc.enable_val
   } else {
       desc.enable_mask
   }.
2. regmap_update_bits(rdev.regmap, desc.enable_reg, desc.enable_mask, val).

`Helpers::disable_regmap(rdev) -> Result<(), Errno>`:
1. let val = if desc.enable_is_inverted {
       if desc.enable_val != 0 { desc.enable_val } else { desc.enable_mask }
   } else {
       desc.disable_val
   }.
2. regmap_update_bits(rdev.regmap, desc.enable_reg, desc.enable_mask, val).

`Helpers::get_voltage_sel_regmap(rdev) -> i32`:
1. regmap_read(rdev.regmap, desc.vsel_reg, &mut val)?.
2. val &= desc.vsel_mask.
3. val >>= ffs(desc.vsel_mask) - 1.
4. return val as i32.

`Helpers::set_voltage_sel_regmap(rdev, sel) -> Result<(), Errno>`:
1. sel <<= ffs(desc.vsel_mask) - 1.
2. regmap_update_bits(rdev.regmap, desc.vsel_reg, desc.vsel_mask, sel)?.
3. if desc.apply_bit != 0:
   - regmap_update_bits(rdev.regmap, desc.apply_reg, desc.apply_bit, desc.apply_bit)?.
4. return Ok(()).

`Helpers::range_selector_to_index(rdev, rval) -> i32`:
1. if desc.linear_range_selectors_bitfield.is_none(): return -EINVAL.
2. rval &= desc.vsel_range_mask.
3. rval >>= ffs(desc.vsel_range_mask) - 1.
4. for i in 0..desc.n_linear_ranges:
   - if desc.linear_range_selectors_bitfield[i] == rval: return i as i32.
5. return -EINVAL.

`Helpers::get_voltage_sel_pickable_regmap(rdev) -> i32`:
1. let r = desc.linear_ranges.ok_or(-EINVAL)?.
2. regmap_read(rdev.regmap, desc.vsel_reg, &mut val)?.
3. regmap_read(rdev.regmap, desc.vsel_range_reg, &mut r_val)?.
4. val &= desc.vsel_mask; val >>= ffs(desc.vsel_mask) - 1.
5. let range = Helpers::range_selector_to_index(rdev, r_val); if range < 0: return -EINVAL.
6. let voltages = linear_range_values_in_range_array(r, range as u32).
7. return (val + voltages) as i32.

`Helpers::set_voltage_sel_pickable_regmap(rdev, sel) -> Result<(), Errno>`:
1. let (mut i, mut voltages_in_range) = (0u32, 0u32).
2. while i < desc.n_linear_ranges:
   - let r = &desc.linear_ranges[i].
   - voltages_in_range = linear_range_values_in_range(r).
   - if sel < voltages_in_range: break.
   - sel -= voltages_in_range.
   - i += 1.
3. if i == desc.n_linear_ranges: return Err(-EINVAL).
4. sel <<= ffs(desc.vsel_mask) - 1.
5. sel += desc.linear_ranges[i].min_sel.
6. let range = (desc.linear_range_selectors_bitfield[i]) << (ffs(desc.vsel_range_mask) - 1).
7. if desc.vsel_reg == desc.vsel_range_reg:
   - regmap_update_bits(rdev.regmap, desc.vsel_reg, desc.vsel_range_mask | desc.vsel_mask, sel | range)?.
8. else:
   - Helpers::write_separate_vsel_and_range(rdev, sel, range)?.
9. if desc.apply_bit != 0:
   - regmap_update_bits(rdev.regmap, desc.apply_reg, desc.apply_bit, desc.apply_bit)?.
10. return Ok(()).

`Helpers::write_separate_vsel_and_range(rdev, sel, range) -> Result<(), Errno>`:
1. let mut range_updated = false.
2. regmap_update_bits_base(rdev.regmap, desc.vsel_range_reg, desc.vsel_range_mask, range, &mut range_updated, false, false)?.
3. if desc.range_applied_by_vsel ∧ range_updated:
   - return regmap_write_bits(rdev.regmap, desc.vsel_reg, desc.vsel_mask, sel).
4. return regmap_update_bits(rdev.regmap, desc.vsel_reg, desc.vsel_mask, sel).

`Helpers::map_voltage_linear(rdev, min_uV, max_uV) -> i32`:
1. if desc.n_voltages == 1 ∧ desc.uV_step == 0:
   - return (min_uV <= desc.min_uV ∧ desc.min_uV <= max_uV) ? 0 : -EINVAL.
2. if desc.uV_step == 0: BUG_ON.
3. let mut min_uV = min_uV.
4. if min_uV < desc.min_uV: min_uV = desc.min_uV.
5. ret = DIV_ROUND_UP(min_uV - desc.min_uV, desc.uV_step) + desc.linear_min_sel.
6. voltage = desc.ops.list_voltage(rdev, ret).
7. if voltage < min_uV ∨ voltage > max_uV: return -EINVAL.
8. return ret.

`Helpers::map_voltage_iterate(rdev, min_uV, max_uV) -> i32`:
1. best_val = i32::MAX; selector = 0.
2. for i in 0..desc.n_voltages:
   - ret = desc.ops.list_voltage(rdev, i); if ret < 0: continue.
   - if ret < best_val ∧ ret >= min_uV ∧ ret <= max_uV: best_val = ret; selector = i.
3. return if best_val != i32::MAX { selector } else { -EINVAL }.

`Helpers::set_current_limit_regmap(rdev, min_uA, max_uA) -> Result<(), Errno>`:
1. let n = desc.n_current_limits; if n == 0: return Err(-EINVAL).
2. let mut sel: i32 = -1.
3. if let Some(t) = desc.curr_table:
   - let ascend = t[n-1] > t[0].
   - if ascend: iterate i = n-1 downto 0; pick first within [min_uA, max_uA]; sel = i; break.
   - else: iterate i = 0..n; pick first within bounds; sel = i; break.
4. if sel < 0: return Err(-EINVAL).
5. sel <<= ffs(desc.csel_mask) - 1.
6. regmap_update_bits(rdev.regmap, desc.csel_reg, desc.csel_mask, sel as u32).

`Helpers::set_ramp_delay_regmap(rdev, ramp_delay) -> Result<(), Errno>`:
1. if desc.n_ramp_values == 0 ∨ desc.ramp_delay_table.is_none(): WARN; return Err(-EINVAL).
2. let mut sel = 0u32.
3. ret = Helpers::find_closest_bigger(ramp_delay, desc.ramp_delay_table, desc.n_ramp_values, &mut sel).
4. if ret != 0: dev_warn("Can't set ramp-delay %u, setting %u").
5. sel <<= ffs(desc.ramp_mask) - 1.
6. regmap_update_bits(rdev.regmap, desc.ramp_reg, desc.ramp_mask, sel).

`Helpers::find_closest_bigger(target, table, num_sel, sel) -> i32`:
1. max = table[0]; maxsel = 0; found = false; tmp = 0.
2. for s in 0..num_sel:
   - if table[s] > max: max = table[s]; maxsel = s.
   - if table[s] >= target:
     - if !found ∨ table[s] - target < tmp - target:
       - tmp = table[s]; *sel = s; found = true.
       - if tmp == target: break.
3. if !found: *sel = maxsel; return -EINVAL.
4. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `selector_round_trip_identity` | INVARIANT | per-set_voltage_sel_regmap then get_voltage_sel_regmap: returns same selector. |
| `enable_inversion_symmetric` | INVARIANT | per-enable_regmap then disable_regmap with inversion: reaches opposite state. |
| `is_enabled_post_enable_is_one` | INVARIANT | per-enable_regmap success ⟹ is_enabled_regmap > 0. |
| `is_enabled_post_disable_is_zero` | INVARIANT | per-disable_regmap success ⟹ is_enabled_regmap returns false. |
| `list_voltage_linear_bounds` | INVARIANT | per-list_voltage_linear: selector >= n_voltages ⟹ -EINVAL. |
| `map_voltage_linear_in_range` | INVARIANT | per-map_voltage_linear success ⟹ list_voltage(rdev, sel) ∈ [min_uV, max_uV]. |
| `pickable_round_trip` | INVARIANT | per-set_pickable then get_pickable: same selector. |
| `find_closest_bigger_finds_min` | INVARIANT | per-find_closest_bigger: result is smallest table entry >= target, or -EINVAL. |
| `current_limit_within_table` | INVARIANT | per-set_current_limit_regmap success ⟹ curr_table[sel] ∈ [min_uA, max_uA]. |
| `vsel_mask_shift_normalized` | INVARIANT | per-get/set_vsel: ffs-1 shifts produce 0-based selector. |
| `bypass_round_trip` | INVARIANT | per-set_bypass(true) ⟹ get_bypass(enable) sets *enable=true. |

### Layer 2: TLA+

`drivers/regulator/helpers.tla`:
- Per-enable/disable + per-vsel get/set + per-pickable + per-list_voltage + per-current-limit + per-ramp.
- Properties:
  - `safety_enable_disable_state_machine` — per-enable then per-disable: returns to initial register state.
  - `safety_vsel_in_mask` — per-set_voltage_sel: written bits are subset of vsel_mask.
  - `safety_list_voltage_bounds` — per-list_voltage_{linear,table,linear_range}: selector OOB ⟹ -EINVAL.
  - `safety_pickable_range_consistency` — per-set_pickable then per-get_pickable: same selector.
  - `safety_apply_bit_after_vsel` — per-set_voltage_sel with apply_bit: apply_reg written after vsel_reg.
  - `liveness_map_voltage_terminates` — per-map_voltage_{iterate,ascend,linear,linear_range}: terminates in bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Helpers::is_enabled_regmap` post: returns i32 according to inversion + enable_val | `Helpers::is_enabled_regmap` |
| `Helpers::enable_regmap` post: writes value matching enable semantics | `Helpers::enable_regmap` |
| `Helpers::set_voltage_sel_regmap` post: vsel_reg has selector ∧ apply_bit if set | `Helpers::set_voltage_sel_regmap` |
| `Helpers::get_voltage_sel_pickable_regmap` post: returns val + linear-range offset ∨ -EINVAL | `Helpers::get_voltage_sel_pickable_regmap` |
| `Helpers::map_voltage_linear` post: list_voltage(ret) ∈ [min_uV, max_uV] | `Helpers::map_voltage_linear` |
| `Helpers::map_voltage_ascend` post: monotonic early-exit correct | `Helpers::map_voltage_ascend` |
| `Helpers::list_voltage_table` post: returns desc.volt_table[selector] or -EINVAL | `Helpers::list_voltage_table` |
| `Helpers::set_current_limit_regmap` post: curr_table[sel] ∈ [min_uA, max_uA] | `Helpers::set_current_limit_regmap` |
| `Helpers::set_ramp_delay_regmap` post: ramp_reg written with closest-bigger selector | `Helpers::set_ramp_delay_regmap` |
| `Helpers::find_closest_bigger` post: result is smallest table[s] >= target ∨ maxsel | `Helpers::find_closest_bigger` |
| `Helpers::is_equal` post: result ⟺ rdev pointer-equality | `Helpers::is_equal` |

### Layer 4: Verus/Creusot functional

`Per-PMIC driver populates regulator_desc → assigns helper as op → core invokes ops(rdev, …) → helper performs regmap_update_bits / regmap_read / linear-range arithmetic → returns selector ↔ uV / errno` semantic equivalence: per-Documentation/power/regulator/regulator.rst + per-include/linux/regulator/driver.h.

## Hardening

(Inherits row-1 features from `drivers/regulator/00-overview.md` § Hardening.)

Regulator-helpers reinforcement:

- **Per-mask-shift via ffs(mask)-1 normalized** — defense against per-driver mask-bit drift.
- **Per-enable_is_inverted symmetric** — defense against per-active-low rail wiring mismatch.
- **Per-enable_val==0 fallback to enable_mask** — defense against per-legacy driver missing enable_val.
- **Per-apply_bit honored after vsel write** — defense against per-PMIC double-buffered voltage latch.
- **Per-list_voltage selector bounds check** — defense against per-OOB volt_table access.
- **Per-linear_min_sel honored** — defense against per-low-selector unreachable region.
- **Per-pickable range_applied_by_vsel** — defense against per-vsel-latches-only-after-range PMIC quirk.
- **Per-map_voltage_linear verify-via-list_voltage** — defense against per-arithmetic-off-by-one ignored.
- **Per-pickable_linear_range overlapping-ranges continues** — defense against per-overlap-skips-valid selector.
- **Per-find_closest_bigger returns largest if no match (with -EINVAL)** — defense against per-out-of-range ramp programmed silently.
- **Per-WARN_ON for missing ramp-delay table** — defense against per-driver-missing-config.
- **Per-BUG_ON for missing volt_table / linear_ranges where required** — defense against per-driver-missing-config (loud).
- **Per-EXPORT_SYMBOL_GPL discipline** — defense against per-GPL-shim out-of-tree.
- **Per-regmap layer atomicity** — defense against per-torn read/write.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY (regmap read/write)** — regulator-helpers are regmap-driven (regulator_enable_regmap, regulator_disable_regmap, regulator_set_voltage_sel_regmap, regulator_get_voltage_sel_regmap, regulator_is_enabled_regmap, regulator_list_voltage_linear_range); regmap's user-facing reflection (/sys/kernel/debug/regmap/<dev>/registers and any sysfs-backed reg-read/reg-write attr) usercopy-whitelisted; defense against per-oversized-register-dump leaking adjacent slab and per-out-of-range register access copying beyond reg_stride * max_register.
- **PAX_KERNEXEC** — regulator_enable_regmap, regulator_disable_regmap, regulator_set_voltage_sel_regmap, regulator_get_voltage_sel_regmap run W^X.
- **PAX_RANDKSTACK** — per-regmap-bus-IRQ (I2C/SPI completion) and per-/sys/class/regulator entry randomize kernel-stack offset.
- **PAX_REFCOUNT** — regmap, regulator_dev refs saturating refcount_t; defense against per-bus-storm refcount overflow.
- **PAX_MEMORY_SANITIZE** — regmap_cache, regulator_linear_range_table, regulator_desc slabs poison-on-free; defense against per-prior-register-cache leak across re-bind of the helper-using driver.
- **PAX_UDEREF** — debugfs regmap reg-read / reg-write enforce split user/kernel on the address+value parse.
- **PAX_RAP/kCFI** — regulator_ops vtable populated by the helpers (enable_regmap, disable_regmap, set_voltage_sel_regmap, get_voltage_sel_regmap, is_enabled_regmap) CFI-protected.
- **GRKERNSEC_HIDESYM** — regulator helpers (regulator_*_regmap, regulator_list_voltage_linear_range) addresses hidden from /proc/kallsyms.
- **GRKERNSEC_DMESG** — "failed to read enable register", "failed to write voltage selector" prints restricted.
- **Per-helper desc->enable_reg / vsel_reg bounded** — every helper checks register against regmap->max_register before bus access; defense against per-DT-typo register write into an unmapped MMIO/I2C slave region.
- **Per-mask bit-width validated** — desc->enable_mask, vsel_mask validated as power-of-two-aligned within reg_stride; defense against per-mask-overlap clobbering adjacent control bits.
- **Per-regmap layer atomicity** — regmap-locked read-modify-write inside *_regmap helpers; defense against per-torn update racing with a foreign driver on the same regmap.
- **Per-EXPORT_SYMBOL_GPL discipline** — all helpers are GPL-only; defense against per-GPL-shim out-of-tree driver linking against regulator_*_regmap.
- **Per-linear-range table immutable-after-init** — regulator_desc->linear_ranges marked const; defense against per-runtime tampering of voltage-selector ↔ microvolt mapping.
- Rationale: regulator-helpers are the regmap-driven canonical implementation of regulator_ops for I2C/SPI PMICs; grsec posture is dominated by PAX_USERCOPY on regmap reflection, register-address / mask bounds at every helper entry, regmap-locked atomicity, CFI on the resulting regulator_ops, and sanitize-on-free of regmap-cache and linear-range tables.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Regulator core registration & consumer API (covered in `drivers/regulator/core.md` Tier-3)
- Fixed-voltage driver (covered in `drivers/regulator/fixed.md` Tier-3)
- Per-vendor PMIC drivers (covered in their respective Tier-3 docs)
- regmap subsystem (covered in `drivers/base/regmap/regmap.md` Tier-3 if expanded)
- linear_range library (covered in `lib/linear_ranges.md` Tier-3 if expanded)
- IRQ chip wiring for over-/under-voltage notifications (covered in `drivers/regulator/irq_helpers.md` Tier-3 if expanded)
- Implementation code
