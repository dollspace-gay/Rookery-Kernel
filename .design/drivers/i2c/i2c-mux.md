# Tier-3: drivers/i2c/i2c-mux.c — I2C multiplexer core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/i2c/00-overview.md
upstream-paths:
  - drivers/i2c/i2c-mux.c (~441 lines)
  - include/linux/i2c-mux.h
  - drivers/i2c/muxes/ (per-mux drivers consuming this core)
  - Documentation/i2c/muxes/ (DT bindings, semantics)
-->

## Summary

An I2C multiplexer fans out a parent I2C bus into a set of mutually-exclusive child segments (channels), each addressable independently while sharing the same physical clock+data lines on the parent side. `drivers/i2c/i2c-mux.c` is the **mux core**: it does not talk to any specific mux chip — it provides the framework that per-mux drivers (`muxes/pca954x`, `muxes/i2c-mux-gpio`, `muxes/i2c-mux-mlxcpld`, `muxes/i2c-arb-gpio-challenge`, ...) use to expose their channels as ordinary `struct i2c_adapter`s. Per allocation: `i2c_mux_alloc(parent, dev, max_adapters, sizeof_priv, flags, select, deselect)` returns a `struct i2c_mux_core` carrying parent+per-channel select/deselect callbacks. Per channel: `i2c_mux_add_adapter(muxc, force_nr, chan_id)` allocates a `struct i2c_mux_priv` (its embedded `struct i2c_adapter` is the virtual child adapter) and registers it. Per transfer on a child adapter: select(chan_id) → __i2c_transfer / __i2c_smbus_xfer on parent → deselect(chan_id). Per locking flavor: `I2C_MUX_LOCKED` (mux-locked: parent's mux_lock serialises all child traffic but parent stays usable concurrently when safe) vs parent-locked (default: lock parent for the whole select+xfer+deselect window, blocks every other consumer). Per arbitrator / gate variants (`I2C_MUX_ARBITRATOR`, `I2C_MUX_GATE`): DT child node naming differs (`i2c-arb`, `i2c-gate`, `i2c-mux`). Per DT walk: scan the matching child node by `reg = <chan_id>` to populate the virtual adapter's `of_node`, so devices declared under that channel enumerate as if on a normal bus. Per ACPI: `acpi_preset_companion` ties the channel to the ACPI sub-node. Per sysfs: each child adapter gets a `mux_device` symlink up to the mux dev, and the mux dev gets `channel-<N>` symlinks down to each child adapter. Critical for: PCA9540/9541/9542/9543/9545/9546/9548/9847 chips, GPIO-driven analog muxes, on-board NXP/Mellanox switch-FPGAs, board-arbitrator gates protecting shared sensors, and every server BMC topology where one host I2C controller fans out to dozens of segments.

This Tier-3 covers `drivers/i2c/i2c-mux.c` (~441 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct i2c_mux_core` | per-mux instance (parent, callbacks, flags, adapters[]) | `I2cMuxCore` |
| `struct i2c_mux_priv` | per-child-adapter (adap, algo, muxc, chan_id) | `I2cMuxPriv` |
| `i2c_mux_alloc()` | per-allocate muxc (devm) | `I2cMux::alloc` |
| `i2c_mux_add_adapter()` | per-register child adapter | `I2cMux::add_adapter` |
| `i2c_mux_del_adapters()` | per-tear-down all children | `I2cMux::del_adapters` |
| `i2c_root_adapter()` | per-walk-to-physical-root | `I2cMux::root_adapter` |
| `__i2c_mux_master_xfer()` | per-mux-locked __i2c_transfer fast path | `I2cMux::master_xfer_atomic_unlocked` |
| `i2c_mux_master_xfer()` | per-mux-locked i2c_transfer | `I2cMux::master_xfer` |
| `__i2c_mux_smbus_xfer()` | per-mux-locked __i2c_smbus_xfer | `I2cMux::smbus_xfer_unlocked` |
| `i2c_mux_smbus_xfer()` | per-mux-locked i2c_smbus_xfer | `I2cMux::smbus_xfer` |
| `i2c_mux_functionality()` | per-functionality forward to parent | `I2cMux::functionality` |
| `i2c_mux_lock_bus` / `_trylock_bus` / `_unlock_bus` | per-mux-locked lock_ops | `I2cMux::lock_ops` |
| `i2c_parent_lock_bus` / `_trylock_bus` / `_unlock_bus` | per-parent-locked lock_ops | `I2cMux::parent_lock_ops` |
| `i2c_mux_lock_ops` | per-lock_operations table (mux-locked) | constant |
| `i2c_parent_lock_ops` | per-lock_operations table (parent-locked) | constant |
| `I2C_MUX_LOCKED` flag | per-mux-locked operating mode | `MuxFlags::Locked` |
| `I2C_MUX_ARBITRATOR` flag | per-arbitrator semantics | `MuxFlags::Arbitrator` |
| `I2C_MUX_GATE` flag | per-gate semantics | `MuxFlags::Gate` |
| `mux_device` sysfs link | per-child → mux backlink | shared |
| `channel-<chan_id>` sysfs link | per-mux → child forward link | shared |

## Compatibility contract

REQ-1: struct i2c_mux_core (per-mux instance):
- parent: per-physical parent `*i2c_adapter`.
- dev: per-`*device` owning the mux (typically the mux driver's i2c_client.dev).
- mux_locked: 1 = use mux_lock_ops (parent's mux_lock + parent i2c_lock_bus only on ROOT_ADAPTER flag); 0 = use parent_lock_ops (always lock parent for select+xfer+deselect).
- arbitrator / gate: per-variant flags driving DT child-node lookup name (`i2c-arb` vs `i2c-gate` vs `i2c-mux`).
- select / deselect: per-channel callbacks `int (*)(struct i2c_mux_core *, u32)` (deselect may be NULL).
- num_adapters / max_adapters: per-bounded array length.
- priv: pointer into the trailing storage allocated for `sizeof_priv` bytes after `adapter[max_adapters]`.
- adapter[]: flexible array of `*i2c_adapter`, one per registered channel.

REQ-2: struct i2c_mux_priv (per-child virtual adapter):
- adap: per-`struct i2c_adapter` exposed to the I2C core; `adap.algo_data == this priv`.
- algo: per-`struct i2c_algorithm` initialised at add-time (xfer / xfer_atomic / smbus_xfer / smbus_xfer_atomic / functionality).
- muxc: backpointer to parent `*i2c_mux_core`.
- chan_id: per-channel identifier passed into select/deselect.

REQ-3: i2c_mux_alloc(parent, dev, max_adapters, sizeof_priv, flags, select, deselect):
- mux_size = struct_size(muxc, adapter, max_adapters).
- muxc = devm_kzalloc(dev, size_add(mux_size, sizeof_priv), GFP_KERNEL); on failure → NULL.
- if sizeof_priv: muxc.priv = &muxc.adapter[max_adapters].
- muxc.parent = parent; muxc.dev = dev.
- muxc.mux_locked = !!(flags & I2C_MUX_LOCKED).
- muxc.arbitrator = !!(flags & I2C_MUX_ARBITRATOR).
- muxc.gate = !!(flags & I2C_MUX_GATE).
- muxc.select = select; muxc.deselect = deselect.
- muxc.max_adapters = max_adapters.
- return muxc.

REQ-4: i2c_mux_add_adapter(muxc, force_nr, chan_id):
- if muxc.num_adapters >= muxc.max_adapters: dev_err "No room"; return -EINVAL.
- priv = kzalloc(sizeof *priv); on failure → -ENOMEM.
- priv.muxc = muxc; priv.chan_id = chan_id.
- /* Wire algo dynamically, forwarding to whichever ops parent supports */
- if parent.algo.master_xfer:
  - priv.algo.xfer = muxc.mux_locked ? i2c_mux_master_xfer : __i2c_mux_master_xfer.
- if parent.algo.master_xfer_atomic: priv.algo.xfer_atomic = priv.algo.master_xfer.
- if parent.algo.smbus_xfer:
  - priv.algo.smbus_xfer = muxc.mux_locked ? i2c_mux_smbus_xfer : __i2c_mux_smbus_xfer.
- if parent.algo.smbus_xfer_atomic: priv.algo.smbus_xfer_atomic = priv.algo.smbus_xfer.
- priv.algo.functionality = i2c_mux_functionality.
- /* Adapter scaffolding */
- snprintf(priv.adap.name, "i2c-%d-mux (chan_id %d)", i2c_adapter_id(parent), chan_id).
- priv.adap.owner = THIS_MODULE; priv.adap.algo = &priv.algo; priv.adap.algo_data = priv.
- priv.adap.dev.parent = &parent.dev.
- priv.adap.retries = parent.retries; priv.adap.timeout = parent.timeout; priv.adap.quirks = parent.quirks.
- priv.adap.lock_ops = muxc.mux_locked ? &i2c_mux_lock_ops : &i2c_parent_lock_ops.
- /* Per-DT child-node lookup */ — see REQ-9.
- /* Per-ACPI companion */ — see REQ-10.
- if force_nr: priv.adap.nr = force_nr; ret = i2c_add_numbered_adapter(&priv.adap).
- else: ret = i2c_add_adapter(&priv.adap).
- on failure: kfree(priv); return ret.
- WARN-on-failure sysfs_create_link(priv.adap.dev.kobj, muxc.dev.kobj, "mux_device").
- WARN-on-failure sysfs_create_link(muxc.dev.kobj, priv.adap.dev.kobj, "channel-<chan_id>").
- dev_info "Added multiplexed i2c bus <id>".
- muxc.adapter[muxc.num_adapters++] = &priv.adap.
- return 0.

REQ-5: i2c_mux_del_adapters(muxc):
- while muxc.num_adapters:
  - adap = muxc.adapter[--muxc.num_adapters].
  - priv = adap.algo_data.
  - np = adap.dev.of_node.
  - muxc.adapter[muxc.num_adapters] = NULL.
  - sysfs_remove_link(muxc.dev.kobj, "channel-<priv.chan_id>").
  - sysfs_remove_link(priv.adap.dev.kobj, "mux_device").
  - i2c_del_adapter(adap).
  - of_node_put(np).
  - kfree(priv).

REQ-6: Per-transfer (mux-locked, master_xfer): i2c_mux_master_xfer(adap, msgs, num):
- priv = adap.algo_data; muxc = priv.muxc; parent = muxc.parent.
- ret = muxc.select(muxc, priv.chan_id).
- if ret >= 0: ret = i2c_transfer(parent, msgs, num).   /* full-stack parent lock acquired here */
- if muxc.deselect: muxc.deselect(muxc, priv.chan_id).   /* unconditional even on select failure */
- return ret.

REQ-7: Per-transfer (parent-locked, __master_xfer): __i2c_mux_master_xfer(adap, msgs, num):
- Identical to REQ-6 except invokes `__i2c_transfer(parent, ...)` — caller (the I2C core lock_ops below) has already taken parent.bus_lock.

REQ-8: Per-SMBus transfer: `[__]i2c_mux_smbus_xfer(adap, addr, flags, read_write, command, size, data)`:
- Same select → [__]i2c_smbus_xfer(parent, ...) → deselect pattern as REQ-6/REQ-7.
- deselect always runs whenever defined, regardless of select / xfer outcome (caller must not see leaked mux state).

REQ-9: Per-DT child-node binding (CONFIG_OF) inside i2c_mux_add_adapter:
- dev_node = muxc.dev.of_node; if !dev_node: skip.
- mux_node =
  - muxc.arbitrator → of_get_child_by_name(dev_node, "i2c-arb").
  - muxc.gate → of_get_child_by_name(dev_node, "i2c-gate").
  - else → of_get_child_by_name(dev_node, "i2c-mux").
- if mux_node ∧ of_property_read_u32(mux_node, "reg", &reg) == 0:
  - /* old-style flat DT entry: mux_node *is* the channel — undo the child-by-name */
  - of_node_put(mux_node); mux_node = NULL.
- if !mux_node: mux_node = of_node_get(dev_node).        /* flat DT: scan dev_node's children */
- else if muxc.arbitrator ∨ muxc.gate: child = of_node_get(mux_node).   /* arb/gate have a single channel == the i2c-arb/i2c-gate node */
- if !child:
  - for_each_child_of_node(mux_node, child):
    - if of_property_read_u32(child, "reg", &reg) != 0: continue.
    - if chan_id == reg: break.
- priv.adap.dev.of_node = child.                         /* refcount held by `child` is transferred to the adapter */
- of_node_put(mux_node).

REQ-10: Per-ACPI companion (CONFIG_ACPI) inside i2c_mux_add_adapter:
- if has_acpi_companion(muxc.dev): acpi_preset_companion(&priv.adap.dev, ACPI_COMPANION(muxc.dev), chan_id).

REQ-11: Per-functionality: i2c_mux_functionality(adap):
- /* The virtual adapter inherits the parent's full functionality vector verbatim */
- return parent.algo.functionality(parent).

REQ-12: Per-lock_operations — mux-locked (i2c_mux_lock_ops):
- lock_bus(adapter, flags):
  - rt_mutex_lock_nested(&parent.mux_lock, i2c_adapter_depth(adapter)).
  - if !(flags & I2C_LOCK_ROOT_ADAPTER): return.       /* mux-internal traffic: don't lock parent */
  - i2c_lock_bus(parent, flags).                       /* root-traffic (e.g. SMBus alert): lock parent too */
- trylock_bus(adapter, flags):
  - if !rt_mutex_trylock(&parent.mux_lock): return 0.
  - if !(flags & I2C_LOCK_ROOT_ADAPTER): return 1.
  - if i2c_trylock_bus(parent, flags): return 1.
  - rt_mutex_unlock(&parent.mux_lock); return 0.
- unlock_bus(adapter, flags):
  - if flags & I2C_LOCK_ROOT_ADAPTER: i2c_unlock_bus(parent, flags).
  - rt_mutex_unlock(&parent.mux_lock).

REQ-13: Per-lock_operations — parent-locked (i2c_parent_lock_ops):
- lock_bus: rt_mutex_lock_nested(parent.mux_lock, i2c_adapter_depth) THEN i2c_lock_bus(parent, flags) — always both.
- trylock_bus: rt_mutex_trylock → on success i2c_trylock_bus(parent) → on failure roll back mux_lock.
- unlock_bus: i2c_unlock_bus(parent) THEN rt_mutex_unlock(parent.mux_lock) — reverse order.

REQ-14: Per-i2c_root_adapter(dev) — walk to physical root:
- Walk dev.parent chain until finding a node whose `type == &i2c_adapter_type`.
- If none: return NULL.
- to_i2c_adapter(node) → i2c_root.
- while i2c_parent_is_i2c_adapter(i2c_root): i2c_root = i2c_parent_is_i2c_adapter(i2c_root).
- return i2c_root (the bottom physical bus, skipping all interposed mux levels).

REQ-15: Per-multi-level nesting:
- A child adapter may itself be the parent of another mux. `i2c_adapter_depth(adapter)` recurses through the chain and yields the lockdep nesting key used by `rt_mutex_lock_nested`. The mux_lock+parent locking dance must remain deadlock-free at any depth.

REQ-16: Per-deselect always runs after select:
- Per REQ-6/7/8: deselect is invoked whenever defined, regardless of the success/failure of either the select() call or the actual transfer. This ensures the mux state is left in a known idle/disconnected position; mux drivers that need fault-on-select to leave the channel selected must implement that policy inside the deselect callback or by passing `deselect == NULL`.

REQ-17: Per-sysfs symlink topology:
- /sys/bus/i2c/devices/i2c-<child>/mux_device → /sys/.../<mux-dev>.
- /sys/.../<mux-dev>/channel-<chan_id> → /sys/bus/i2c/devices/i2c-<child>.
- Failures are warned (`WARN(...)`), not fatal.

REQ-18: Per-naming:
- priv.adap.name = "i2c-<parent_id>-mux (chan_id <N>)" — i2c-tools recognise this prefix to render topology trees (e.g., `i2cdetect -l` output).

## Acceptance Criteria

- [ ] AC-1: `i2c_mux_alloc(parent, dev, N, sizeof_priv, 0, sel, desel)` returns muxc with parent / select / deselect / max_adapters set; muxc.priv lives in trailing storage when sizeof_priv > 0.
- [ ] AC-2: `i2c_mux_add_adapter(muxc, 0, K)` registers a virtual adapter whose name is `i2c-<P>-mux (chan_id K)`, `.dev.parent == &parent.dev`, `.lock_ops` chosen per `mux_locked`.
- [ ] AC-3: Per-channel transfer on the virtual adapter calls select(K) before parent transfer and deselect(K) after, regardless of transfer outcome.
- [ ] AC-4: `master_xfer_atomic` / `smbus_xfer_atomic` propagated only when the parent supports them.
- [ ] AC-5: Mux-locked + non-ROOT lock_bus flag: parent bus_lock NOT taken; concurrent traffic on parent's other consumers proceeds (subject to mux_lock).
- [ ] AC-6: Mux-locked + ROOT_ADAPTER lock_bus flag: both mux_lock and parent bus_lock held.
- [ ] AC-7: Parent-locked: both mux_lock and parent bus_lock held for the entire select+xfer+deselect window.
- [ ] AC-8: DT path: `i2c-mux { mux@N { reg = <N>; ... } }` populates virtual adapter's of_node from the matching child; devices under that node enumerate.
- [ ] AC-9: DT arbitrator/gate variant uses `i2c-arb` / `i2c-gate` child-node name; single channel == the named node itself.
- [ ] AC-10: Old-style flat DT (mux children directly under dev_node with `reg = <N>`) supported via the "mux_node has reg" fallback.
- [ ] AC-11: ACPI: child adapter's ACPI companion set via `acpi_preset_companion(..., chan_id)`.
- [ ] AC-12: `force_nr != 0`: child adapter registered with that bus number; failure path frees priv.
- [ ] AC-13: sysfs `mux_device` symlink on child + `channel-<N>` on mux exist after registration; both removed by `i2c_mux_del_adapters`.
- [ ] AC-14: `i2c_mux_del_adapters` releases all children in reverse order, removes both symlinks per channel, calls `i2c_del_adapter`, drops of_node, frees priv.
- [ ] AC-15: `i2c_root_adapter(client.dev)` returns the lowermost physical i2c_adapter (skipping all mux levels) for any device on a multi-level mux tree.

## Architecture

```
struct I2cMuxCore {
  parent: *I2cAdapter,
  dev: *Device,
  mux_locked: bool,
  arbitrator: bool,
  gate: bool,
  select: fn(*I2cMuxCore, u32) -> i32,
  deselect: Option<fn(*I2cMuxCore, u32) -> i32>,
  num_adapters: u32,
  max_adapters: u32,
  priv: *u8,                            // points past adapter[max_adapters]
  adapter: [*I2cAdapter; max_adapters], // flexible array tail
}

struct I2cMuxPriv {
  adap: I2cAdapter,                     // the virtual child adapter
  algo: I2cAlgorithm,                   // populated per-add to match parent's ops
  muxc: *I2cMuxCore,
  chan_id: u32,
}
```

`I2cMux::alloc(parent, dev, max_adapters, sizeof_priv, flags, select, deselect) -> Option<*I2cMuxCore>`:
1. mux_size = struct_size(I2cMuxCore, adapter, max_adapters).
2. muxc = devm_kzalloc(dev, mux_size + sizeof_priv, GFP_KERNEL); on null → None.
3. if sizeof_priv > 0: muxc.priv = &muxc.adapter[max_adapters].
4. muxc.parent = parent; muxc.dev = dev.
5. muxc.mux_locked = flags & I2C_MUX_LOCKED.
6. muxc.arbitrator = flags & I2C_MUX_ARBITRATOR.
7. muxc.gate = flags & I2C_MUX_GATE.
8. muxc.select = select; muxc.deselect = deselect.
9. muxc.max_adapters = max_adapters.
10. return Some(muxc).

`I2cMux::add_adapter(muxc, force_nr, chan_id) -> i32`:
1. if muxc.num_adapters >= muxc.max_adapters: return -EINVAL.
2. priv = kzalloc(I2cMuxPriv); on null → -ENOMEM.
3. priv.muxc = muxc; priv.chan_id = chan_id.
4. /* algo selection */
5. if muxc.parent.algo.master_xfer:
   - priv.algo.xfer = if muxc.mux_locked { i2c_mux_master_xfer } else { __i2c_mux_master_xfer }.
6. if muxc.parent.algo.master_xfer_atomic: priv.algo.xfer_atomic = priv.algo.master_xfer.
7. if muxc.parent.algo.smbus_xfer:
   - priv.algo.smbus_xfer = if muxc.mux_locked { i2c_mux_smbus_xfer } else { __i2c_mux_smbus_xfer }.
8. if muxc.parent.algo.smbus_xfer_atomic: priv.algo.smbus_xfer_atomic = priv.algo.smbus_xfer.
9. priv.algo.functionality = i2c_mux_functionality.
10. /* adapter wiring */
11. priv.adap.name = format!("i2c-{}-mux (chan_id {})", muxc.parent.id, chan_id).
12. priv.adap.owner = THIS_MODULE.
13. priv.adap.algo = &priv.algo; priv.adap.algo_data = priv.
14. priv.adap.dev.parent = &muxc.parent.dev.
15. priv.adap.retries = muxc.parent.retries; priv.adap.timeout = muxc.parent.timeout; priv.adap.quirks = muxc.parent.quirks.
16. priv.adap.lock_ops = if muxc.mux_locked { &i2c_mux_lock_ops } else { &i2c_parent_lock_ops }.
17. /* DT child-node resolution: REQ-9 */
18. /* ACPI companion: REQ-10 */
19. ret = if force_nr { i2c_add_numbered_adapter(&priv.adap) } else { i2c_add_adapter(&priv.adap) }.
20. if ret < 0: kfree(priv); return ret.
21. sysfs_create_link("mux_device" : priv.adap.dev → muxc.dev).
22. sysfs_create_link("channel-<N>" : muxc.dev → priv.adap.dev).
23. dev_info "Added multiplexed i2c bus <id>".
24. muxc.adapter[muxc.num_adapters++] = &priv.adap.
25. return 0.

`I2cMux::del_adapters(muxc)`:
1. while muxc.num_adapters > 0:
   - n = --muxc.num_adapters.
   - adap = muxc.adapter[n]; priv = adap.algo_data; np = adap.dev.of_node.
   - muxc.adapter[n] = NULL.
   - sysfs_remove_link(muxc.dev, "channel-<priv.chan_id>").
   - sysfs_remove_link(priv.adap.dev, "mux_device").
   - i2c_del_adapter(adap).
   - of_node_put(np).
   - kfree(priv).

`I2cMux::xfer_via_select(muxc, chan_id, body)` — generalised inner of REQ-6/7/8:
1. ret = (muxc.select)(muxc, chan_id).
2. if ret >= 0: ret = body(muxc.parent).
3. if let Some(d) = muxc.deselect: d(muxc, chan_id).
4. return ret.

`I2cMux::root_adapter(dev) -> Option<*I2cAdapter>`:
1. walk dev.parent until i2c.type == &i2c_adapter_type or null.
2. if null: return None.
3. root = to_i2c_adapter(i2c).
4. while let Some(p) = i2c_parent_is_i2c_adapter(root): root = p.
5. return Some(root).

`I2cMux::lock_ops` (mux-locked):
- lock_bus: rt_mutex_lock_nested(parent.mux_lock, depth(self)); if I2C_LOCK_ROOT_ADAPTER: i2c_lock_bus(parent).
- trylock_bus: rt_mutex_trylock(parent.mux_lock) → if !ROOT: 1; else i2c_trylock_bus(parent) ? 1 : roll-back.
- unlock_bus: if ROOT: i2c_unlock_bus(parent); rt_mutex_unlock(parent.mux_lock).

`I2cMux::parent_lock_ops` (parent-locked):
- lock_bus: rt_mutex_lock_nested(parent.mux_lock, depth(self)); i2c_lock_bus(parent, flags).
- trylock_bus: rt_mutex_trylock(parent.mux_lock); i2c_trylock_bus(parent) ? 1 : roll-back.
- unlock_bus: i2c_unlock_bus(parent); rt_mutex_unlock(parent.mux_lock).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `select_deselect_balanced` | INVARIANT | per-xfer: if select succeeds, deselect runs exactly once (when defined); if select fails, deselect still runs (when defined). |
| `add_adapter_oom_no_partial` | INVARIANT | per-add_adapter: on kzalloc / i2c_add_*_adapter failure, no leaked priv and num_adapters unchanged. |
| `del_adapter_reverse_order` | INVARIANT | per-del_adapters: drains num_adapters → 0 with no double-free. |
| `mux_locked_no_parent_lock_on_internal` | INVARIANT | per-mux-locked lock_bus(non-ROOT): parent.bus_lock NOT held. |
| `mux_locked_root_locks_both` | INVARIANT | per-mux-locked lock_bus(ROOT): both mux_lock + parent.bus_lock held. |
| `parent_locked_locks_both` | INVARIANT | per-parent-locked lock_bus: both held for entire xfer window. |
| `trylock_rollback_on_failure` | INVARIANT | per-trylock_bus: parent-trylock failure → mux_lock released. |
| `unlock_order_inverse_of_lock` | INVARIANT | per-unlock_bus: parent unlocked before mux_lock (avoids inversion). |
| `of_node_refcount_balanced` | INVARIANT | per-add: of_node_get on `child` matched by of_node_put in del; mux_node always put. |
| `chan_id_bounds` | INVARIANT | per-add: num_adapters < max_adapters before increment. |

### Layer 2: TLA+

`drivers/i2c/i2c-mux.tla`:
- Per-mux-instance + per-channel-set + per-xfer + per-deselect lifecycle.
- Actors: child adapter consumers, mux-internal callers, root-adapter callers, registration/deregistration thread.
- Properties:
  - `safety_no_overlap_distinct_channels` — per-mux-locked: at most one chan_id selected on the parent bus at any instant.
  - `safety_select_implies_deselect_eventually` — per-xfer: select(K) is followed by deselect(K) (when defined) regardless of failure.
  - `safety_lock_inversion_free` — per-multi-level mux: nested rt_mutex_lock_nested with i2c_adapter_depth is acyclic.
  - `safety_no_root_lock_skip` — per-ROOT_ADAPTER flag: parent bus_lock taken whenever flag present.
  - `liveness_xfer_terminates` — per-child xfer: select+xfer+deselect completes (no infinite spin under contention).
  - `liveness_del_drains_to_zero` — per-del_adapters: num_adapters reaches 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `I2cMux::alloc` post: returned muxc has the requested parent, dev, callbacks, max_adapters; priv null iff sizeof_priv == 0 | `I2cMux::alloc` |
| `I2cMux::add_adapter` post: on ret == 0, muxc.adapter[old_n] is the new priv.adap; on ret != 0, no state change | `I2cMux::add_adapter` |
| `I2cMux::del_adapters` post: muxc.num_adapters == 0 | `I2cMux::del_adapters` |
| `I2cMux::xfer_via_select` post: deselect called when defined and select was attempted | `I2cMux::xfer_via_select` |
| `I2cMux::root_adapter` post: returned adapter has no i2c-adapter ancestor | `I2cMux::root_adapter` |
| `mux_lock_ops` post: invariants per REQ-12 | `i2c_mux_lock_ops` |
| `parent_lock_ops` post: invariants per REQ-13 | `i2c_parent_lock_ops` |

### Layer 4: Verus/Creusot functional

`Per-mux setup → per-channel add (with DT/ACPI binding) → per-xfer (select → parent ops → deselect) → per-locking-mode → per-teardown` semantic equivalence vs upstream — verified against Documentation/i2c/muxes/ and the in-tree per-mux drivers (pca954x, i2c-mux-gpio, i2c-mux-mlxcpld, i2c-arb-gpio-challenge) as ground-truth consumers.

## Hardening

(Inherits row-1 features from `drivers/i2c/00-overview.md` § Hardening.)

I2C-mux reinforcement:

- **Per-deselect always-runs** — defense against per-channel-leak (a stuck mux leaving a child connected after a fault would mis-route subsequent root-adapter transfers).
- **Per-select failure still deselects** — defense against partial-state stuck mux.
- **Per-max_adapters bound enforced** — defense against per-add overrun of the flexible adapter[] array.
- **Per-rt_mutex_lock_nested with i2c_adapter_depth** — defense against per-multi-level mux lock inversion (lockdep-validated nesting).
- **Per-trylock rollback on partial acquisition** — defense against per-mux_lock leak on parent-lock-fail.
- **Per-unlock ordering parent-then-mux** — defense against per-lock-order regression.
- **Per-mux_locked flag honoured** — defense against per-spurious-parent-lock denial-of-service on shared parents (sensors blocked while unrelated mux channel transfers).
- **Per-DT child-node refcount strict get/put** — defense against per-OF node UAF on mux teardown.
- **Per-ACPI companion scoped to chan_id** — defense against per-ACPI cross-channel device binding.
- **Per-sysfs symlink creation WARN-only** — defense against per-symlink-failure regression refusing mux registration on an otherwise-functional system.
- **Per-functionality forward from parent** — defense against per-virtual-adapter advertising features the parent cannot actually deliver.
- **Per-priv kzalloc'd** — defense against per-stale-stack residue in algo_data dispatch.
- **Per-of_node_put on del_adapters** — defense against per-leak in module-unload cycle.

## Open Questions

- Should the mux core grow an explicit invariant that the `select` callback is idempotent w.r.t. the currently-selected channel? Several mux drivers already short-circuit redundant selects; promoting this to the core would let parent-locked transfers skip the bus-quiet window. Out of scope for parity; track for post-parity optimisation.
- The current `deselect = NULL` convention (mux stays on last channel) is widely used but undocumented in `<linux/i2c-mux.h>`. Worth adding a Rookery doc-comment on the trait.

## Out of Scope

- Per-mux chip drivers (pca954x, gpio, mlxcpld, demux-pinctrl, etc.) — covered by `drivers/i2c/muxes/*.md` Tier-3 docs.
- I2C core (adapter / client / driver lifecycle, lock_bus / __i2c_transfer / i2c_smbus_xfer) — covered by `drivers/i2c/core-base.md`.
- I2C SMBus alert / host-notify — covered by `drivers/i2c/i2c-smbus.md`.
- I2C address translator (i2c-atr.c) — covered by `drivers/i2c/atr.md`.
- /dev/i2c-N chardev — covered by `drivers/i2c/dev.md`.
- Implementation code.
