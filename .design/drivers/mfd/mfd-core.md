# Tier-3: drivers/mfd/mfd-core.c — MFD (Multi-Function Device) core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/mfd/00-overview.md
upstream-paths:
  - drivers/mfd/mfd-core.c (~455 lines)
  - include/linux/mfd/core.h
-->

## Summary

A **Multi-Function Device** (MFD) is a single physical device (typically a PMIC, codec, or SoC companion chip) that exposes several logical sub-functions (regulator, RTC, GPIO, watchdog, charger, audio, ...) behind a shared transport (I2C, SPI, MMIO). The MFD core translates one parent device + an array of `struct mfd_cell` descriptors into a forest of child `struct platform_device` instances, one per sub-function, so each sub-function can be bound by its own standard subsystem driver. Per-`mfd_add_devices(parent, id, cells, n_devs, mem_base, irq_base, domain)` allocates and registers child platform devices; per-child resource translation rewrites cell-relative `IORESOURCE_MEM` offsets against the parent's `mem_base` and `IORESOURCE_IRQ` numbers against either `irq_base` or an `irq_domain` mapping. Per-`mfd_remove_devices(parent)` walks the parent's children in reverse and unregisters all `mfd_dev_type` platform devices. Per-`devm_mfd_add_devices` ties the lifetime of the whole child set to the parent's devres unbind. Critical for: PMIC/codec/SoC-companion drivers, sub-function modularity, OF/ACPI fwnode propagation to children.

This Tier-3 covers `drivers/mfd/mfd-core.c` (~455 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mfd_cell` | per-sub-function descriptor | `MfdCell` |
| `struct mfd_cell_acpi_match` | per-ACPI _HID/_ADR match for cell | `MfdCellAcpiMatch` |
| `struct mfd_of_node_entry` | per-OF-node↔child binding record | `MfdOfNodeEntry` |
| `mfd_dev_type` | `device_type` tag identifying MFD children | `MFD_DEV_TYPE` |
| `mfd_add_devices()` | per-batch register children | `MfdCore::add_devices` |
| `mfd_add_device()` (static) | per-child build + register | `MfdCore::add_device` |
| `mfd_remove_devices()` | per-batch unregister children | `MfdCore::remove_devices` |
| `mfd_remove_devices_late()` | per-batch unregister at MFD_DEP_LEVEL_HIGH | `MfdCore::remove_devices_late` |
| `mfd_remove_devices_fn()` (static) | per-child unregister callback | `MfdCore::remove_devices_fn` |
| `devm_mfd_add_devices()` | per-devres-managed batch add | `MfdCore::devm_add_devices` |
| `devm_mfd_dev_release()` (static) | per-devres release callback | `MfdCore::devm_release` |
| `mfd_match_of_node_to_dev()` (static) | per-OF-child match + reservation | `MfdCore::match_of_node_to_dev` |
| `mfd_acpi_add_device()` (static) | per-ACPI fwnode attach | `MfdCore::acpi_add_device` |
| `mfd_get_cell()` | per-pdev cell retrieve | `MfdCore::get_cell` |
| `MFD_CELL_BASIC` macro | per-cell convenience constructor | `mfd_cell_basic!` |
| `MFD_DEP_LEVEL_NORMAL` / `_HIGH` | per-removal-tier marker | `MfdDepLevel::{Normal,High}` |
| `PLATFORM_DEVID_AUTO` | per-platform-API auto-id sentinel | `PLATFORM_DEVID_AUTO` |

## Compatibility contract

REQ-1: struct mfd_cell:
- name: per-child platform driver name (matches `platform_driver.driver.name`).
- id: per-child id offset (added to parent's `id` unless `PLATFORM_DEVID_AUTO`).
- level: per-removal-tier — `MFD_DEP_LEVEL_NORMAL` (default 0) or `MFD_DEP_LEVEL_HIGH`.
- enable / disable / suspend / resume: optional cell-level callbacks (legacy; called by driver, not core).
- platform_data + pdata_size: per-cell `platform_data` blob copied into pdev.
- swnode: per-cell software-fwnode handle.
- of_compatible: per-cell `compatible` string for DT child match.
- of_reg: per-cell first-reg address to disambiguate DT children with same compatible.
- use_of_reg: per-cell flag enabling `of_reg` matching.
- acpi_match: per-cell ACPI `_HID`/`_CID`/`_ADR` matcher.
- resources + num_resources: per-cell resource template (parent-relative for MEM; cell-relative for IRQ).
- ignore_resource_conflicts: per-cell bypass for `acpi_check_resource_conflict`.
- pm_runtime_no_callbacks: per-cell PM-runtime hint.
- parent_supplies + num_parent_supplies: per-cell regulator-bulk supply aliases.

REQ-2: mfd_add_devices(parent, id, cells, n_devs, mem_base, irq_base, domain):
- for i in 0..n_devs:
  - ret = mfd_add_device(parent, id, &cells[i], mem_base, irq_base, domain).
  - if ret: mfd_remove_devices(parent); return ret.
- return 0.

REQ-3: mfd_add_device(parent, id, cell, mem_base, irq_base, domain):
- /* Compute platform-id */
- platform_id = (id == PLATFORM_DEVID_AUTO) ? PLATFORM_DEVID_AUTO : id + cell.id.
- /* Allocate pdev */
- pdev = platform_device_alloc(cell.name, platform_id).
- /* Stash cell-copy on pdev */
- pdev.mfd_cell = kmemdup(cell, sizeof *cell).
- /* Allocate temp resource array */
- res = kcalloc(cell.num_resources, sizeof *res).
- /* Inherit parent fields */
- pdev.dev.parent = parent.
- pdev.dev.type = &mfd_dev_type.
- pdev.dev.dma_mask = parent.dma_mask.
- pdev.dev.dma_parms = parent.dma_parms.
- pdev.dev.coherent_dma_mask = parent.coherent_dma_mask.
- /* Regulator supply aliases */
- regulator_bulk_register_supply_alias(&pdev.dev, cell.parent_supplies, parent, cell.parent_supplies, cell.num_parent_supplies).
- /* OF child match (if CONFIG_OF ∧ parent.of_node ∧ cell.of_compatible) */
- for_each_child_of_node(parent.of_node, np):
  - if of_device_is_compatible(np, cell.of_compatible):
    - if !of_device_is_available(np): disabled=true; continue.
    - ret = mfd_match_of_node_to_dev(pdev, np, cell).
    - if ret == -EAGAIN: continue.
    - if ret: goto fail.
    - break.
- if no match ∧ disabled: ret=0; goto fail (cleanly skip disabled).
- /* ACPI fwnode attach (if CONFIG_ACPI) */
- mfd_acpi_add_device(cell, pdev).
- /* platform_data */
- if cell.pdata_size: platform_device_add_data(pdev, cell.platform_data, cell.pdata_size).
- /* swnode */
- if cell.swnode: device_add_software_node(&pdev.dev, cell.swnode).
- /* Resource translation */
- for r in 0..cell.num_resources:
  - res[r].name = cell.resources[r].name.
  - res[r].flags = cell.resources[r].flags.
  - if (flags & IORESOURCE_MEM) ∧ mem_base:
    - res[r].parent = mem_base.
    - res[r].start = mem_base.start + cell.resources[r].start.
    - res[r].end   = mem_base.start + cell.resources[r].end.
  - elif (flags & IORESOURCE_IRQ):
    - if domain:
      - WARN_ON(start != end) /* single-IRQ mappings only */.
      - res[r].start = res[r].end = irq_create_mapping(domain, cell.resources[r].start).
    - else:
      - res[r].start = irq_base + cell.resources[r].start.
      - res[r].end   = irq_base + cell.resources[r].end.
  - else:
    - res[r].{parent,start,end} = cell.resources[r].{parent,start,end}.
  - if !cell.ignore_resource_conflicts ∧ has_acpi_companion(pdev): acpi_check_resource_conflict(&res[r]).
- platform_device_add_resources(pdev, res, cell.num_resources).
- platform_device_add(pdev).
- if cell.pm_runtime_no_callbacks: pm_runtime_no_callbacks(&pdev.dev).
- kfree(res).
- return 0.

REQ-4: MEM-resource translation:
- Cell author writes `.start/.end` relative to the parent's MEM window (offset within `mem_base`).
- Core rewrites them to absolute addresses by adding `mem_base->start`.
- `res[r].parent = mem_base` so the child's region is sibling-registered under the parent's region (standard `iomem_resource` tree discipline).
- When `mem_base == NULL` ∨ resource is not `IORESOURCE_MEM`, the cell's fields are copied through verbatim (escape hatch for absolute MEM addrs).

REQ-5: IRQ-resource translation:
- Two modes: legacy `irq_base` offset OR `irq_domain` mapping.
- `irq_base` mode: `res.start = irq_base + cell.resources[r].start` (cell numbers are local).
- `domain` mode: `irq_create_mapping(domain, cell.resources[r].start)` returns a Linux IRQ for the per-cell hwirq; `start == end` is required (no IRQ-ranges through a domain).
- Driver consumes its child's IRQ via `platform_get_irq()` like any platform device.

REQ-6: mfd_match_of_node_to_dev(pdev, np, cell):
- /* Reserve each DT child to at most one pdev */
- scoped_guard(mutex, &mfd_of_node_mutex):
  - for of_entry in mfd_of_node_list: if of_entry.np == np: return -EAGAIN.
- if !cell.use_of_reg: skip address-disambiguation; go allocate.
- of_property_read_reg(np, 0, &addr, NULL); if cell.of_reg != addr: return -EAGAIN.
- /* allocate */
- of_entry = kzalloc(sizeof *of_entry).
- of_entry.dev = &pdev.dev; of_entry.np = of_node_get(np).
- list_add_tail(&of_entry.list, &mfd_of_node_list).
- device_set_node(&pdev.dev, of_fwnode_handle(np)).
- return 0.

REQ-7: mfd_acpi_add_device(cell, pdev) (CONFIG_ACPI):
- parent = ACPI_COMPANION(pdev.dev.parent); if !parent: return.
- if cell.acpi_match.pnpid:
  - acpi_dev_for_each_child(parent, match_device_ids(_HID/_CID), &wd).
  - adev = wd.adev.
- elif cell.acpi_match.adr:
  - adev = acpi_find_child_device(parent, match.adr, false).
- set_primary_fwnode(&pdev.dev, acpi_fwnode_handle(adev ?: parent)).
  - Uses `set_primary_fwnode` (NOT `device_set_node`) so an OF fwnode attached earlier is not overwritten; ACPI takes precedence when both are present.

REQ-8: mfd_remove_devices(parent):
- level = MFD_DEP_LEVEL_NORMAL.
- device_for_each_child_reverse(parent, &level, mfd_remove_devices_fn).
- — Reverse order undoes registration order (children that depend on earlier children unbind first).

REQ-9: mfd_remove_devices_late(parent):
- level = MFD_DEP_LEVEL_HIGH.
- device_for_each_child_reverse(parent, &level, mfd_remove_devices_fn).
- — Use case: parent driver must perform power-down sequencing where some children (typically regulators) outlive all others.

REQ-10: mfd_remove_devices_fn(dev, &level):
- if dev.type != &mfd_dev_type: return 0.
- pdev = to_platform_device(dev); cell = mfd_get_cell(pdev).
- if level ∧ cell.level > *level: return 0  /* defer to _late() */.
- if cell.swnode: device_remove_software_node(&pdev.dev).
- under mfd_of_node_mutex: drop any mfd_of_node_list entries whose dev == &pdev.dev.
- regulator_bulk_unregister_supply_alias(dev, cell.parent_supplies, cell.num_parent_supplies).
- platform_device_unregister(pdev).

REQ-11: devm_mfd_add_devices(dev, id, cells, n_devs, mem_base, irq_base, domain):
- ptr = devres_alloc(devm_mfd_dev_release, sizeof *ptr, GFP_KERNEL).
- ret = mfd_add_devices(dev, id, cells, n_devs, mem_base, irq_base, domain).
- if ret < 0: devres_free(ptr); return ret.
- *ptr = dev; devres_add(dev, ptr).
- — On parent unbind, devm_mfd_dev_release(dev, res) → mfd_remove_devices(dev).

REQ-12: MFD_CELL_BASIC(_name, _res, _pdata, _pdsize, _id):
- Compile-time initializer expanding to:
  - .name = _name, .platform_data = _pdata, .pdata_size = _pdsize, .id = _id, .resources = _res, .num_resources = ARRAY_SIZE(_res).
- Sister macros: `MFD_CELL_NAME(_name)`, `MFD_CELL_RES(_name, _res)`, `MFD_CELL_OF(...)`, `OF_MFD_CELL(...)`, `OF_MFD_CELL_REG(...)`.

REQ-13: cell-copy ownership:
- `pdev.mfd_cell = kmemdup(cell, sizeof *cell)` — each child owns a private copy so the parent's `cells[]` array may be `const` / stack / unloadable.
- `mfd_get_cell(pdev)` returns that copy.
- Freed via `platform_device_release` when pdev refcnt drops to 0.

REQ-14: mfd_of_node_list discipline:
- Global list + global mutex (mfd_of_node_mutex).
- Each entry binds one DT child node to one pdev.
- Prevents a single DT node from being claimed twice when two cells share `of_compatible` and differ only by `of_reg`.

REQ-15: Resource-conflict checking:
- For ACPI-companion children, every translated resource is fed through `acpi_check_resource_conflict()` before `platform_device_add_resources()`.
- Cells with `ignore_resource_conflicts = true` bypass.
- Failure is non-fatal at registration but returned to caller.

REQ-16: fwnode policy:
- OF children: `device_set_node()` (full set, including secondary).
- ACPI children: `set_primary_fwnode()` (only primary; preserves OF as secondary).
- Plain children with neither: pdev fwnode left untouched (inherits nothing automatically).

REQ-17: Error-rollback:
- mfd_add_device labels (in unwind order): fail_res_conflict → fail_of_entry → fail_alias → fail_res → fail_device → fail_alloc.
- mfd_add_devices: on i-th failure calls mfd_remove_devices(parent) (which sees i previously-added children).

## Acceptance Criteria

- [ ] AC-1: mfd_add_devices(n_devs): n_devs child platform devices appear under parent with `dev.type == &mfd_dev_type`.
- [ ] AC-2: Each child's MEM resources are translated: child[r].start == mem_base.start + cell.resources[r].start.
- [ ] AC-3: Each child's IRQ resources via `irq_base`: child[r].start == irq_base + cell.resources[r].start.
- [ ] AC-4: Each child's IRQ resources via `domain`: child[r].start == irq_create_mapping(domain, cell.resources[r].start) and start == end.
- [ ] AC-5: cell.platform_data of size pdata_size is copied onto child pdev (platform_device_add_data).
- [ ] AC-6: cell.swnode is attached and removed on unregister.
- [ ] AC-7: OF child with matching compatible and (if use_of_reg) of_reg is found and registered to mfd_of_node_list exactly once.
- [ ] AC-8: Two cells claiming the same DT compatible with different of_reg each bind their own DT node — no node is bound twice.
- [ ] AC-9: ACPI companion: child's primary fwnode set to matched acpi_device (or parent if no _HID/_ADR match).
- [ ] AC-10: mfd_remove_devices(parent): unregisters all `mfd_dev_type` children in reverse order.
- [ ] AC-11: mfd_remove_devices() does NOT remove children with cell.level == MFD_DEP_LEVEL_HIGH; mfd_remove_devices_late() removes them.
- [ ] AC-12: devm_mfd_add_devices(parent): on parent unbind, all children are removed.
- [ ] AC-13: Mid-batch failure: mfd_add_devices rewinds already-registered children before returning the error.
- [ ] AC-14: Cell with cell.id offset: child platform_id = (id + cell.id); with PLATFORM_DEVID_AUTO: platform-API allocates.
- [ ] AC-15: Disabled DT child (`status="disabled"`): cell silently skipped, no pdev created, no error.

## Architecture

```
struct MfdCell {
  name: &'static str,
  id: i32,
  level: u8,                                 // MFD_DEP_LEVEL_{NORMAL,HIGH}
  platform_data: *const u8, pdata_size: usize,
  swnode: Option<&'static SoftwareNode>,
  of_compatible: Option<&'static str>,
  of_reg: u64,
  use_of_reg: bool,
  acpi_match: Option<&'static MfdCellAcpiMatch>,
  resources: &'static [Resource],
  num_resources: usize,
  ignore_resource_conflicts: bool,
  pm_runtime_no_callbacks: bool,
  parent_supplies: *const *const c_char,
  num_parent_supplies: usize,
}

struct MfdOfNodeEntry { dev: *Device, np: *DeviceNode, list: ListHead }

static MFD_OF_NODE_LIST: Mutex<LinkedList<MfdOfNodeEntry>> = ...;
static MFD_DEV_TYPE: DeviceType = DeviceType { name: "mfd_device" };
```

`MfdCore::add_devices(parent, id, cells, mem_base, irq_base, domain) -> Result<()>`:
1. for (i, cell) in cells.iter().enumerate():
   - MfdCore::add_device(parent, id, cell, mem_base, irq_base, domain)?.
2. on Err at i: MfdCore::remove_devices(parent); return Err.
3. return Ok.

`MfdCore::add_device(parent, id, cell, mem_base, irq_base, domain) -> Result<()>`:
1. platform_id = if id == PLATFORM_DEVID_AUTO { PLATFORM_DEVID_AUTO } else { id + cell.id }.
2. pdev = platform_device_alloc(cell.name, platform_id)?.
3. pdev.mfd_cell = kmemdup(cell)?.
4. res = kcalloc::<Resource>(cell.num_resources)?.
5. pdev.dev.parent = parent.
6. pdev.dev.type = &MFD_DEV_TYPE.
7. pdev.dev.dma_mask = parent.dma_mask.
8. pdev.dev.dma_parms = parent.dma_parms.
9. pdev.dev.coherent_dma_mask = parent.coherent_dma_mask.
10. regulator_bulk_register_supply_alias(&pdev.dev, cell.parent_supplies, parent, cell.parent_supplies, cell.num_parent_supplies)?.
11. /* OF */ if cfg!(CONFIG_OF) ∧ parent.of_node.is_some() ∧ cell.of_compatible.is_some():
    - for np in parent.of_node.children():
      - if of_device_is_compatible(np, cell.of_compatible):
        - if !of_device_is_available(np): disabled = true; continue.
        - match MfdCore::match_of_node_to_dev(pdev, np, cell):
          - Err(EAGAIN): continue.
          - Err(e): goto fail_alias.
          - Ok: break.
    - if no match ∧ disabled: return Ok (cleanly skipped).
12. MfdCore::acpi_add_device(cell, pdev).
13. if cell.pdata_size > 0: platform_device_add_data(pdev, cell.platform_data, cell.pdata_size)?.
14. if let Some(swnode) = cell.swnode: device_add_software_node(&pdev.dev, swnode)?.
15. /* Resource translation */
16. for r in 0..cell.num_resources:
    - res[r].name = cell.resources[r].name.
    - res[r].flags = cell.resources[r].flags.
    - if (flags & IORESOURCE_MEM) ∧ mem_base.is_some():
      - res[r].parent = mem_base.
      - res[r].start = mem_base.start + cell.resources[r].start.
      - res[r].end   = mem_base.start + cell.resources[r].end.
    - elif (flags & IORESOURCE_IRQ):
      - if let Some(d) = domain:
        - WARN_ON(cell.resources[r].start != cell.resources[r].end).
        - hwirq = cell.resources[r].start.
        - virq = irq_create_mapping(d, hwirq).
        - res[r].start = virq; res[r].end = virq.
      - else:
        - res[r].start = irq_base + cell.resources[r].start.
        - res[r].end   = irq_base + cell.resources[r].end.
    - else:
      - res[r].parent = cell.resources[r].parent.
      - res[r].start  = cell.resources[r].start.
      - res[r].end    = cell.resources[r].end.
    - if !cell.ignore_resource_conflicts ∧ has_acpi_companion(&pdev.dev):
      - acpi_check_resource_conflict(&res[r])?.
17. platform_device_add_resources(pdev, &res, cell.num_resources)?.
18. platform_device_add(pdev)?.
19. if cell.pm_runtime_no_callbacks: pm_runtime_no_callbacks(&pdev.dev).
20. kfree(res); return Ok.

`MfdCore::match_of_node_to_dev(pdev, np, cell) -> Result<()>`:
1. { let _g = MFD_OF_NODE_LIST.lock();
   for of_entry in &*_g: if of_entry.np == np: return Err(EAGAIN); }
2. if cell.use_of_reg:
   - of_property_read_reg(np, 0, &mut addr, NULL)? else Err(EAGAIN).
   - if cell.of_reg != addr: return Err(EAGAIN).
3. of_entry = kzalloc::<MfdOfNodeEntry>()?.
4. of_entry.dev = &pdev.dev; of_entry.np = of_node_get(np).
5. MFD_OF_NODE_LIST.lock().push_back(of_entry).
6. device_set_node(&pdev.dev, of_fwnode_handle(np)).
7. Ok.

`MfdCore::acpi_add_device(cell, pdev)`:
1. parent_adev = ACPI_COMPANION(pdev.dev.parent); if !parent_adev: return.
2. adev = None.
3. if let Some(m) = cell.acpi_match:
   - if let Some(pnpid) = m.pnpid:
     - ids[0].id = pnpid; ids[1] = {0}.
     - wd = { ids: &ids, adev: None }.
     - acpi_dev_for_each_child(parent_adev, match_device_ids, &wd).
     - adev = wd.adev.
   - else:
     - adev = acpi_find_child_device(parent_adev, m.adr, false).
4. set_primary_fwnode(&pdev.dev, acpi_fwnode_handle(adev.unwrap_or(parent_adev))).

`MfdCore::remove_devices(parent)`:
1. level = MFD_DEP_LEVEL_NORMAL.
2. device_for_each_child_reverse(parent, &level, MfdCore::remove_devices_fn).

`MfdCore::remove_devices_late(parent)`:
1. level = MFD_DEP_LEVEL_HIGH.
2. device_for_each_child_reverse(parent, &level, MfdCore::remove_devices_fn).

`MfdCore::remove_devices_fn(dev, &level) -> i32`:
1. if dev.type != &MFD_DEV_TYPE: return 0.
2. pdev = to_platform_device(dev).
3. cell = mfd_get_cell(pdev).
4. if cell.level > level: return 0 (defer to _late).
5. if cell.swnode.is_some(): device_remove_software_node(&pdev.dev).
6. { let _g = MFD_OF_NODE_LIST.lock();
     retain(|e| e.dev != &pdev.dev else { of_node_put(e.np); free(e); false }); }
7. regulator_bulk_unregister_supply_alias(dev, cell.parent_supplies, cell.num_parent_supplies).
8. platform_device_unregister(pdev).
9. return 0.

`MfdCore::devm_add_devices(dev, id, cells, mem_base, irq_base, domain) -> Result<()>`:
1. ptr = devres_alloc(MfdCore::devm_release, sizeof *Device)?.
2. MfdCore::add_devices(dev, id, cells, mem_base, irq_base, domain).map_err(|e| { devres_free(ptr); e })?.
3. *ptr = dev.
4. devres_add(dev, ptr).
5. Ok.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mem_resource_translation_correct` | INVARIANT | per-add_device: for IORESOURCE_MEM ∧ mem_base: res.start == mem_base.start + cell.start ∧ res.end == mem_base.start + cell.end. |
| `irq_base_translation_correct` | INVARIANT | per-add_device: for IORESOURCE_IRQ ∧ !domain: res.start == irq_base + cell.start. |
| `irq_domain_singleton` | INVARIANT | per-add_device: for IORESOURCE_IRQ ∧ domain: cell.start == cell.end. |
| `disabled_dt_cleanly_skipped` | INVARIANT | per-add_device: if all matching DT nodes have status="disabled": returns Ok, no pdev registered. |
| `of_node_unique_binding` | INVARIANT | per-mfd_of_node_list: no DT np appears in two entries. |
| `reverse_unregistration_order` | INVARIANT | per-remove_devices: device_for_each_child_reverse used (LIFO of registration). |
| `level_high_deferred_to_late` | INVARIANT | per-remove_devices_fn: cell.level > *level ⟹ skipped. |
| `mfd_cell_copy_owned` | INVARIANT | per-pdev: pdev.mfd_cell freshly kmemdup'd, not aliasing caller's array. |
| `devm_release_invokes_remove_devices` | INVARIANT | per-devm: devres release ⟹ mfd_remove_devices(dev). |
| `acpi_fwnode_no_overwrite_of` | INVARIANT | per-acpi_add_device: uses set_primary_fwnode, not device_set_node. |

### Layer 2: TLA+

`drivers/mfd/mfd-core.tla`:
- Per-add_devices batch + per-resource-translation + per-OF-reservation + per-remove sequencing.
- Properties:
  - `safety_batch_atomicity` — mfd_add_devices: success ⟹ all n_devs registered; failure ⟹ none remain registered (rolled back).
  - `safety_no_double_of_binding` — at all times, each DT np appears in mfd_of_node_list at most once.
  - `safety_reverse_remove_order` — remove_devices unbinds children strictly in reverse of registration.
  - `safety_level_high_two_phase` — children with level=HIGH are not removed by remove_devices but are removed by remove_devices_late.
  - `liveness_devm_release_completes` — on parent unbind, MfdCore::remove_devices eventually invoked and terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MfdCore::add_device` post: success ⟹ pdev registered ∧ resources translated ∧ fwnode set | `MfdCore::add_device` |
| `MfdCore::add_devices` post: success ⟹ all cells registered; on failure ⟹ no leftover children | `MfdCore::add_devices` |
| `MfdCore::match_of_node_to_dev` post: success ⟹ MFD_OF_NODE_LIST extended by exactly one entry | `MfdCore::match_of_node_to_dev` |
| `MfdCore::remove_devices_fn` post: cell.level ≤ *level ⟹ pdev unregistered ∧ of_entry purged | `MfdCore::remove_devices_fn` |
| `MfdCore::acpi_add_device` post: primary fwnode points to matched ACPI device or parent (never overwrites OF secondary) | `MfdCore::acpi_add_device` |
| `MfdCore::devm_add_devices` post: success ⟹ devres callback armed on dev | `MfdCore::devm_add_devices` |

### Layer 4: Verus/Creusot functional

`Per-MFD-parent invocation: mfd_add_devices(cells) → forest of platform_devices with translated resources → children probe → mfd_remove_devices() reverses` semantic equivalence: per-Documentation/driver-api/mfd.rst.

## Hardening

(Inherits row-1 features from `drivers/mfd/00-overview.md` § Hardening.)

MFD-core reinforcement:

- **Per-cell copy via kmemdup** — defense against per-caller-array stack/unload UAF.
- **Per-OF-node single-binding mutex** — defense against per-DT-node double-claim races.
- **Per-resource translation bounds-checked vs mem_base** — defense against per-cell-overflow MMIO escape (cell.end + mem_base.start must fit).
- **Per-IRQ-domain singleton enforcement (WARN_ON start != end)** — defense against per-bogus-range hwirq mapping.
- **Per-acpi_check_resource_conflict** — defense against per-firmware-overlap clobber.
- **Per-set_primary_fwnode (not device_set_node) on ACPI path** — defense against per-OF-fwnode overwrite when both ACPI and DT describe the same device.
- **Per-reverse-removal order** — defense against per-LIFO dependency unbinding (a child that supplies a clock to another child must outlive it).
- **Per-MFD_DEP_LEVEL_HIGH two-phase removal** — defense against per-regulator-pulled-during-shutdown-of-consumer.
- **Per-devm_mfd_add_devices** — defense against per-parent-error-path child-leak.
- **Per-mfd_of_node_list cleanup on remove** — defense against per-DT-node stale-reservation after rebind.
- **Per-regulator-supply-alias paired register/unregister** — defense against per-alias-leak on unbind.
- **Per-mid-batch rollback** — defense against per-partial-registration zombie children.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Individual MFD parent drivers (e.g. tps6586x, axp20x, wm831x, cros_ec) — separate Tier-3 each
- drivers/base/platform.c platform_device API (covered in `drivers/base/platform.md` Tier-3)
- drivers/base/regmap/* (covered separately if expanded)
- drivers/base/regulator/core.c regulator_bulk_*_supply_alias (covered in `drivers/regulator/core.md`)
- drivers/of/ DT parsing primitives (covered in `drivers/of/*.md`)
- drivers/acpi/ ACPI device walking (covered in `drivers/acpi/*.md`)
- kernel/irq/irqdomain.c (covered in `kernel/irq/irqdomain.md` Tier-3)
- Implementation code
