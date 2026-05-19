# Tier-3: drivers/dma/dmaengine.c + virt-dma.c — DMA framework core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/dma/00-overview.md
upstream-paths:
  - drivers/dma/dmaengine.c
  - drivers/dma/dmaengine.h
  - drivers/dma/virt-dma.c
  - drivers/dma/virt-dma.h
  - include/linux/dmaengine.h
-->

## Summary

`dmaengine.c` (~1637 lines) is the framework spine: it owns the global engine list, the per-class device hierarchy, the channel allocator with capability + NUMA matching, the OF/ACPI client-facing helpers (`dma_request_chan`), and per-channel sysfs/debugfs. It does *not* program HW directly — every per-engine driver supplies `device_prep_*`, `device_issue_pending`, `device_tx_status`, `device_config`, `device_terminate_all` callbacks that this file dispatches to.

`virt-dma.c` (~143 lines) is the small but ubiquitous helper that drivers like `pl330`, `at_xdmac`, `imx-sdma`, `mediatek/*`, and `bcm2835-dma` build on: it provides three lists (`desc_submitted` / `desc_issued` / `desc_completed`) plus a `tasklet_struct` so per-driver IRQ handlers only have to call `vchan_cookie_complete()` and the framework drains completion callbacks off-IRQ.

This Tier-3 covers `dma_async_device_register`, channel allocation, the `dma_chan_get`/`dma_chan_put` refcount dance, and the virt-channel completion pipeline.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `dma_async_device_register(dev)` | validate caps, assign id, register per-channel kobjs | `DmaDevice::register` |
| `dma_async_device_unregister(dev)` | tear down on driver remove | `DmaDevice::unregister` |
| `dma_async_device_channel_register(dev, chan, name)` | per-channel late-register | `DmaDevice::register_channel` |
| `__dma_request_channel(mask, fn, param, np)` | filter-fn allocator | `Subsystem::request_channel` |
| `dma_request_chan(dev, name)` | OF/ACPI named-channel resolver | `Subsystem::request_chan_by_name` |
| `dma_request_chan_by_mask(mask)` | cap-only allocator | `Subsystem::request_chan_by_mask` |
| `dma_release_channel(chan)` | release client reference | `DmaChan::release` |
| `dma_chan_get(chan)` / `dma_chan_put(chan)` | per-channel refcount + module pin | `DmaChan::get` / `_put` |
| `dma_channel_rebalance()` | recompute per-CPU `channel_table[]` after register/unregister | `Subsystem::rebalance` |
| `dma_find_channel(tx_type)` | pick a globally-published channel per `enum dma_transaction_type` | `Subsystem::find_channel` |
| `dma_issue_pending_all()` | broadcast-issue across all engines (for `async_tx` framework) | `Subsystem::issue_pending_all` |
| `dma_sync_wait(chan, cookie)` | spin until cookie complete | `DmaChan::sync_wait` |
| `vchan_init(vc, dmadev)` | helper: init `virt_dma_chan` lists + tasklet | `VirtChan::init` |
| `vchan_tx_submit(tx)` | move desc to `desc_submitted` | `VirtChan::submit` |
| `vchan_tx_desc_free(tx)` | free unstarted desc | `VirtChan::desc_free` |
| `vchan_complete(t)` | tasklet: drain `desc_completed`, invoke callbacks | `VirtChan::complete_tasklet` |
| `vchan_find_desc(vc, cookie)` | lookup live descriptor by cookie | `VirtChan::find_desc` |

## Compatibility contract

REQ-1: `dma_async_device_register` rejects an engine where any bit set in `cap_mask` lacks the corresponding `device_prep_*` callback; partial registration is forbidden and returns `-EIO`.

REQ-2: Per-engine `dev_id` is allocated from a global `dma_ida` IDA; channels become `dma%dchan%d` named devices under `class/dma/`.

REQ-3: `dma_chan_get` order: `try_module_get(owner)` -> `device_alloc_chan_resources(chan)` -> `client_count++` -> `balance_ref_count(chan)`. Failure at any step unwinds the previous; resource alloc failure releases the module ref.

REQ-4: `dma_chan_put` order: `client_count--` -> if zero `device_free_chan_resources(chan)` + `module_put(owner)`; also drops one ref from `dmaengine_ref_count` if it was a "general-purpose" allocation.

REQ-5: `dma_channel_rebalance` rebuilds `channel_table[]` (per-CPU, per-`enum dma_transaction_type` pointer to best channel) after every register/unregister; uses `min_chan(cap, cpu)` to prefer NUMA-local, then least-loaded.

REQ-6: `__dma_request_channel` accepts a filter function + param so per-DT-binding drivers can disambiguate channels with identical caps; filter return false skips the candidate.

REQ-7: `dma_request_chan(dev, name)` first tries `of_dma_request_slave_channel(dev->of_node, name)`, then ACPI `acpi_dma_request_slave_chan_by_name`, then the legacy named-slave-map; on no match returns `-EPROBE_DEFER` to allow driver-load ordering.

REQ-8: virt-channel: every descriptor enqueued via `vchan_tx_submit` has its cookie assigned by `dma_cookie_assign(tx)`; cookies are per-channel monotonically increasing 32-bit (wrap handled by `dma_cookie_status`).

REQ-9: virt-channel: `vchan_complete` tasklet runs at TASKLET_PRIO; per-tasklet invocation drains the full `desc_completed` list under per-`vc->lock`, but invokes each `tx->callback` with the lock dropped.

## Acceptance Criteria

- [ ] AC-1: `dmatest` end-to-end test passes on all registered engines (per-engine MEMCPY + SG + XOR if cap supports).
- [ ] AC-2: Unregister-with-active-clients returns `-EBUSY`; unregister-with-no-clients succeeds and module ref hits zero.
- [ ] AC-3: NUMA: on a 2-node host with engines on both nodes, `dma_request_chan_by_mask` from a CPU on node 0 prefers node-0 engine via `dma_channel_rebalance`.
- [ ] AC-4: virt-channel stress: 1M descriptors submitted with random callbacks complete with zero leaks (verified by slab-account + `vc->desc_completed` empty at teardown).
- [ ] AC-5: cookie wraparound: after `2**32` submissions, `dma_cookie_status` correctly orders `last_completed` vs `last_used`.

## Architecture

`DmaDevice::register` validates caps then walks the channels list under `dma_list_mutex` (write):

1. Validate `cap_mask` vs callbacks (`DMA_MEMCPY` -> `device_prep_dma_memcpy`, etc.).
2. `ida_alloc(&dma_ida)` -> `dev_id`.
3. For each `dma_chan` in `channels`: assign `chan_id`, init `kobj`, call `device_register(&chan->dev->device)`.
4. `list_add_tail_rcu(&device->global_node, &dma_device_list)`.
5. If `dma_has_cap(DMA_PRIVATE, mask)` skip rebalance; else `dma_channel_rebalance()`.
6. Register debugfs `summary` entry.

`Subsystem::request_channel(mask, fn, param, np)`:

1. Acquire `dma_list_mutex` (read-rcu).
2. For each `dma_device` satisfying `mask`: walk channels, call `private_candidate(mask, device, fn, param)` to find a channel with `client_count == 0` (private channels are exclusive).
3. On hit: drop mutex, `dma_chan_get(chan)`. If `-ENOMEM` from `device_alloc_chan_resources`, retry next candidate.

`DmaChan::get(chan)`:

```c
int dma_chan_get(struct dma_chan *chan) {
  struct module *owner = dma_chan_to_owner(chan);
  int ret;
  if (chan->client_count) { chan->client_count++; return 0; }
  if (!try_module_get(owner)) return -ENODEV;
  if (chan->device->device_alloc_chan_resources) {
    ret = chan->device->device_alloc_chan_resources(chan);
    if (ret < 0) { module_put(owner); return ret; }
  }
  if (!dma_has_cap(DMA_PRIVATE, chan->device->cap_mask))
    balance_ref_count(chan);
  chan->client_count++;
  return 0;
}
```

VirtChan completion path (per-engine driver IRQ -> framework):

1. Per-engine IRQ handler iterates HW completion ring.
2. For each completed HW descriptor: lookup `virt_dma_desc` (driver-private linkage), call `vchan_cookie_complete(&vd->tx)`.
3. `vchan_cookie_complete` (in `virt-dma.h`): under `vc->lock`, set `vc->cyclic = NULL` (if applicable), `list_move_tail(&vd->node, &vc->desc_completed)`, `dma_cookie_complete(&vd->tx)`, `tasklet_schedule(&vc->task)`.
4. `vchan_complete(t)` (tasklet): under `vc->lock`, splice `desc_completed` into local list, drop lock, iterate calling `tx->callback`/`tx->callback_result` and `vd->vc->desc_free(vd)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `chan_refcount_no_uaf` | UAF | `dma_chan_put` cannot release engine while `client_count > 0` on any per-engine channel |
| `cookie_monotonic` | INVARIANT | per-channel cookie strictly increases until wrap; `dma_cookie_status` handles wrap correctly |
| `private_chan_exclusive` | UNIQUENESS | a channel allocated via `__dma_request_channel` is unreachable to `dma_find_channel` |
| `vchan_desc_no_double_free` | UAF | a `virt_dma_desc` enters `desc_completed` exactly once between submit and free |

### Layer 2: TLA+

`models/dmaengine/vchan_completion.tla` (parent-declared): proves that concurrent submit + IRQ completion + tasklet drain preserve FIFO per-channel order, that callback invocation always occurs after the corresponding HW write to the completion ring is visible, and that no descriptor is double-callbacked.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| Post `dma_async_device_register`: every channel reachable via `dma_find_channel` for any cap bit in `mask` | `DmaDevice::register` |
| Post `dma_chan_get`: `client_count >= 1` AND `try_module_get(owner)` returned true | `DmaChan::get` |
| Post `dma_chan_put` when `client_count == 0`: `device_free_chan_resources` called AND `module_put(owner)` called exactly once | `DmaChan::put` |
| Post `vchan_complete` tasklet: `desc_completed` empty AND each descriptor's `desc_free` called exactly once | `VirtChan::complete_tasklet` |

### Layer 4: Verus functional

Encoded as: client `dma_request_chan` -> `dma_async_issue_pending` -> HW IRQ -> `vchan_cookie_complete` -> tasklet `vchan_complete` -> `tx->callback` invoked AND descriptor freed AND `client_count` reflecting `dma_release_channel` -> module-unbind safe.

## Hardening

`dma_async_device_register` validates cap-bit-to-callback consistency but does **not** validate that the callback typesig matches `enum dma_transaction_type`'s expected signature — kCFI fills that gap. `cap_mask` itself is a bitmap with `DMA_TX_TYPE_END` bits; pre-validate range so future stack-allocated mask buffers cannot OOB.

`dma_chan_get` runs `try_module_get` -> `device_alloc_chan_resources` -> `client_count++` without holding `dma_list_mutex` across resource alloc (alloc can sleep). This opens a race where unregister can begin between `try_module_get` and `client_count++`; `dma_async_device_unregister` MUST wait for `client_count == 0` *and* `chan->device->chancnt`-equivalent quiescence under RCU. Today this relies on `synchronize_rcu` in unregister; document the boundary.

`dmaengine_ref_count` is a `long` not `atomic_long_t` — protected by `dma_list_mutex`. Refcount overflow there is functionally impossible but `PAX_REFCOUNT` policy still demands `atomic_t` + saturate; migrate.

Virt-channel `vc->lock` is a plain spinlock; `vchan_complete` drops it before calling client callbacks. Client callback invoking `dmaengine_terminate_all` is documented-safe only because the callback runs outside `vc->lock`; a wedge here (e.g. callback grabbing `vc->lock` directly) deadlocks. The boundary is delicate; capture as Verus precondition.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `dma_chan`, `dma_device`, `virt_dma_chan`, and per-channel sysfs string buffers; reject copy_to/from_user on raw pointer fields like `chan->private`.
- **PAX_KERNEXEC** — `device_alloc_chan_resources`, `device_prep_dma_*`, `device_issue_pending`, `device_tx_status`, `device_pause`, `device_terminate_all` placed in `__ro_after_init`; tasklet handler `vchan_complete` text RO.
- **PAX_RANDKSTACK** — randomize stack across `dma_async_device_register`, `__dma_request_channel`, `dma_chan_get`/`_put`, `vchan_complete`.
- **PAX_REFCOUNT** — saturating refcount on `dma_chan.client_count` AND `dmaengine_ref_count` (migrate to `atomic_long_t` with saturating helpers); overflow trap defeats register/unregister race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free on `dma_chan` device kobj backing + per-channel HW resource pools; sanitize completed `virt_dma_desc` before re-pool to drop source/dest DMA addresses.
- **PAX_UDEREF** — SMAP/PAN on dmatest debugfs entries and on any future uapi taking client buffer pointers.
- **PAX_RAP / kCFI** — `device_prep_dma_memcpy`/`_sg`/`_xor`/`_pq`/`_cyclic`/`_slave_sg`/`_interleaved_dma` indirect-call sites typed; `tx->tx_submit`, `tx->callback`, `tx->callback_result`, `tx->desc_free` typed; vtable in `__ro_after_init`.
- **GRKERNSEC_HIDESYM** — `chan->private`, `chan->device`, and per-channel debugfs pointer disclosure gated by CAP_SYSLOG.
- **GRKERNSEC_DMESG** — engine probe banners (often disclosing PCI BDF, MMIO base, IRQ vectors) restricted to CAP_SYSLOG.
- **`dma_async_device_register` kCFI** — refuse registration if any `device_prep_*` typesig drifts from `enum dma_transaction_type` expectation.
- **`dma_cap_mask` validated** — every cap-bit set in `cap_mask` cross-checked against non-NULL callback; future expansion guarded by `DMA_TX_TYPE_END` bound.
- **SG-list bounded** — `dma_prep_slave_sg` SG-entry count clamped to `dma_dev->max_sg_burst`; defense against per-driver SG-overflow buffer escapes.
- **`dma_ida` ID aging** — IDs not reused for at least one RCU grace period after `dma_async_device_unregister`; stale `chan->device->dev_id` references fault rather than alias.

Rationale: dmaengine is a *plurality* — sixty-plus engine drivers ride this spine and any one bad `device_prep_*` callback can corrupt the framework. Concentrating discipline at register (kCFI typesig validation, cap-mask sanity, SG bounds) lifts the per-driver bar without touching every driver. Refcount saturation, zero-on-free, and ID aging close the lifecycle races that engine churn (hot-add NICs, USB-DMA, GPU-DMA hot-bind) makes routine.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-engine drivers (IDXD in `idxd.md`, IOAT future, PL330 future, etc.)
- OF/ACPI binding internals (covered in `of_dma.c` / `acpi_dma.c` future Tier-3s)
- `async_tx` framework (covered under `crypto/async_tx/` future Tier-3)
- 32-bit-only paths
- Implementation code
