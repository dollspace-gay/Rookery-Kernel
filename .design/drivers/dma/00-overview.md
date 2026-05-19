# Tier-3: drivers/dma/ — DMA Engine subsystem overview

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/dma/00-overview.md
upstream-paths:
  - drivers/dma/
  - drivers/dma/dmaengine.c
  - drivers/dma/dmaengine.h
  - drivers/dma/virt-dma.c
  - drivers/dma/virt-dma.h
  - include/linux/dmaengine.h
-->

## Summary

`drivers/dma/` hosts the kernel's hardware-DMA (memory-to-memory / memory-to-peripheral / peripheral-to-peripheral) engine framework. Distinct from `dma-mapping`/`iommu` (which programs IOVA translations for DMA-master devices), this subsystem brokers *async copy/transform offload engines* — IOAT, Intel IDXD (DSA/IAA), Xilinx AXIDMA, ARM PL08x/PL330, Apple ADMAC, Atmel AT_XDMAC, Renesas DMAC, MediaTek HSDMA, Marvell mv_xor, Broadcom SBA-RAID, Synopsys DesignWare, NXP DPAA2-QDMA, Freescale eDMA, and dozens more.

Clients allocate a channel (`dma_chan`), build a transaction descriptor (`dma_async_tx_descriptor`), submit it, and either spin on the cookie or attach a completion callback. The framework provides device registration, channel allocator with NUMA + capability matching, refcounted channel ownership, OF/ACPI helpers, and the **virtual channel** (`virt-dma`) helper pattern that most modern drivers use to multiplex N software descriptors over M HW channels with a tasklet completion path.

This Tier-3 overview maps the core surface; per-driver internals live in `dmaengine-core.md`, `idxd.md`, and future siblings.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dma_device` | per-engine descriptor: channels + caps + ops | `drivers::dmaengine::DmaDevice` |
| `struct dma_chan` | per-channel handle: client_count + refcount + table_count | `drivers::dmaengine::DmaChan` |
| `struct dma_async_tx_descriptor` | per-transaction submit + cookie + callback | `drivers::dmaengine::TxDescriptor` |
| `enum dma_transaction_type` | DMA_MEMCPY / DMA_SG / DMA_XOR / DMA_PQ / DMA_CYCLIC / DMA_SLAVE / ... | `drivers::dmaengine::TxType` |
| `dma_async_device_register(dev)` / `_unregister(dev)` | per-engine register/unregister | `DmaDevice::register` / `_unregister` |
| `dma_request_chan(dev, name)` / `__dma_request_channel(mask, fn, param, np)` | client channel allocator | `Subsystem::request_chan` |
| `dma_release_channel(chan)` | release client reference | `DmaChan::release` |
| `dma_chan_get(chan)` / `dma_chan_put(chan)` | refcount client_count + module ref | `DmaChan::get` / `_put` |
| `struct virt_dma_chan` / `vchan_init(vc, dmadev)` | virt-channel helper: list-of-submitted / issued / active / completed | `drivers::dmaengine::VirtChan` |
| `vchan_tx_submit(tx)` / `vchan_tx_desc_free(tx)` / `vchan_complete(t)` | virt-channel submit + free + tasklet | `VirtChan::submit` / `_complete` |

## Per-driver-family map

| Family | Driver dir | Personality |
|---|---|---|
| Intel IOAT | `ioat/` | mem-to-mem DMA on PCH/server NB; legacy XOR/PQ for RAID |
| Intel IDXD | `idxd/` | DSA / IAA accelerators (kernel + user ENQCMDS); see `idxd.md` |
| Synopsys DesignWare | `dw/`, `dw-axi-dmac/`, `dw-edma/` | embedded SoC DMAC, PCIe eDMA |
| ARM PrimeCell | `amba-pl08x.c` | PL080/PL081 SoC DMAC |
| Atmel | `at_hdmac.c`, `at_xdmac.c` | SAMA5 family DMA |
| Freescale / NXP | `fsldma.c`, `fsl-edma-*.c`, `fsl-qdma.c`, `fsl-dpaa2-qdma/` | eDMA + Queue Manager |
| Marvell | `mv_xor.c`, `mv_xor_v2.c` | XOR+memcpy on Armada |
| Broadcom | `bcm-sba-raid.c`, `bcm2835-dma.c` | RPi-class DMA, SBA-RAID accelerator |
| Apple | `apple-admac.c` | M1/M2 Audio DMAC |
| MediaTek | `mediatek/` | HSDMA + UART-DMA |
| Loongson / Renesas / HiSi / Img | `loongson/`, `hisi_dma.c`, `img-mdc-dma.c` | per-SoC engines |
| Test harness | `dmatest.c` | per-engine MEMCPY/SG/XOR self-test |
| Virt-chan helper | `virt-dma.c` | shared substrate for most modern drivers |

## Compatibility contract

REQ-1: Per-engine `dma_device` carries `cap_mask: dma_cap_mask_t` (bitmap of `enum dma_transaction_type`); allocator only returns channels whose owning device satisfies a mask.

REQ-2: Per-channel `client_count` tracks outstanding `dma_chan_get`s; channels with `client_count == 0` are reclaimable; `dma_async_device_unregister` is permitted only when every per-device channel hits `client_count == 0`.

REQ-3: Per-engine NUMA affinity: `dma_chan_is_local(chan, cpu)` checks `dev_to_node(chan->device->dev) == cpu_to_node(cpu)`; `min_chan(cap, cpu)` prefers the local engine, falling back to least-loaded global.

REQ-4: Per-transaction lifecycle: `device_prep_dma_*` returns descriptor; client sets `tx->callback`/`tx->callback_result`; `tx->tx_submit(tx)` returns cookie; client calls `dma_async_issue_pending(chan)` to fire HW.

REQ-5: Per-driver virt-chan helper: descriptors progress `vc->desc_submitted` -> `_issued` -> `_completed` lists; HW completion IRQ enqueues to `_completed`, tasklet drains list invoking per-tx callback.

REQ-6: OF/ACPI binding: `of_dma_request_slave_channel`/`acpi_dma_request_slave_chan_by_name` resolves named channel from DT/ACPI table to a registered `dma_device`.

REQ-7: Per-channel sysfs: `/sys/class/dma/dmaXchanY/{memcpy_count,bytes_transferred,in_use}` accounting; debugfs root `/sys/kernel/debug/dmaengine/summary` enumerates active engines + clients.

REQ-8: Per-cookie completion polling: `dma_async_is_tx_complete(chan, cookie, &last, &used)` returns `DMA_COMPLETE`/`DMA_IN_PROGRESS`/`DMA_PAUSED`/`DMA_ERROR`; `dma_sync_wait(chan, cookie)` is the blocking variant.

REQ-9: Module refcount: per-channel `module_get(chan_to_owner(chan))` taken on first client; released on last `dma_chan_put`; defeats unbind while in use.

## Acceptance Criteria

- [ ] AC-1: `cat /sys/kernel/debug/dmaengine/summary` enumerates registered engines + per-channel client name on systems with IOAT / IDXD / PL330 / etc.
- [ ] AC-2: `dmatest` self-test in `drivers/dma/dmatest.c` runs N concurrent DMA_MEMCPY transactions to completion across all registered engines.
- [ ] AC-3: I2S / SPI / UART peripheral with DMA backing (e.g. PL011 + PL08x) routes RX/TX through dmaengine and reports correct byte counts via per-channel sysfs `bytes_transferred`.
- [ ] AC-4: Hot-unbind of a `dma_device` with no clients succeeds; hot-unbind with client_count > 0 returns `-EBUSY`.
- [ ] AC-5: OF binding: a peripheral DT node with `dmas`/`dma-names` resolves via `dma_request_chan(dev, "tx")` to the right channel.

## Architecture

The framework keeps a global `LIST_HEAD(dma_device_list)` guarded by `dma_list_mutex`; per-engine drivers populate `dma_device` (channels[], cap_mask, device_alloc_chan_resources, device_free_chan_resources, device_prep_dma_*, device_issue_pending, device_tx_status, device_config, device_pause, device_resume, device_terminate_all, device_synchronize) then call `dma_async_device_register`. The framework validates that any cap bit set in `cap_mask` has a corresponding `device_prep_*` function pointer non-NULL (refusing partial impls), allocates a per-device ID via `dma_ida`, registers each `dma_chan` as a child device under `dma_devclass`, then exposes the engine to clients.

Client allocator `__dma_request_channel(mask, filter_fn, filter_param, np)` walks `dma_device_list` under the mutex, calls `private_candidate(mask, device, filter_fn, filter_param)` on each, and on first hit calls `dma_chan_get(chan)` which: (a) `try_module_get(owner)`, (b) calls `chan->device->device_alloc_chan_resources(chan)` to let the driver allocate HW state (descriptor pool, IRQ, ring buffers), (c) increments `client_count`, (d) `balance_ref_count(chan)` reclaims excess module refs left over from earlier `dmaengine_get` global pin.

Virt-DMA pattern (`virt-dma.c`): each HW channel wraps a `virt_dma_chan` containing `desc_submitted`, `desc_issued`, `desc_completed`, plus a `tasklet_struct`. Driver `tx_submit` calls `vchan_tx_submit(tx)` which moves the descriptor from "free" to `desc_submitted`. Driver `device_issue_pending` calls `vchan_start_descriptor(vc)` to splice `desc_submitted` -> `desc_issued` and starts HW. IRQ handler calls `vchan_cookie_complete(&desc->tx)` to splice into `desc_completed` and schedules the tasklet. `vchan_complete(t)` drains `desc_completed`, invokes callbacks, frees descriptors via `desc_free`.

The per-CPU `channel_table[DMA_TX_TYPE_END]` published by `dma_channel_rebalance` is the fast-path lookup for `dma_find_channel(tx_type)` used by the `async_tx` framework (RAID, crypto offload). It is recomputed after every register/unregister/rebalance event; NUMA-locality-first then least-loaded.

Per-driver classification: most older drivers (IOAT, fsldma, hisi_dma, mv_xor) hand-roll their own descriptor management; modern drivers (PL330, AT_XDMAC, IMX-SDMA, MediaTek HSDMA, RCAR-DMAC) build on the virt-chan helper and let the framework do completion sequencing. IDXD is its own beast (cross-ref `idxd.md`) because it exposes both kernel `dma_device` paths and userspace ENQCMDS portals.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `engine_no_dangling_after_unregister` | UAF | post-`dma_async_device_unregister` no `dma_chan*` reachable via `dma_find_channel` |
| `client_count_no_underflow` | INVARIANT | `dma_chan_put` cannot drive `client_count` below zero |
| `vchan_lists_disjoint` | INVARIANT | per-`virt_dma_chan` a descriptor sits in exactly one of `desc_submitted`/`_issued`/`_active`/`_completed`/`_terminated` |

### Layer 2: TLA+

`models/dmaengine/lifecycle.tla` (parent-declared): proves engine register / client get / client put / engine unregister sequence preserves `client_count >= 0` and `module_ref >= 0` under concurrent N-client races.

## Hardening

Per-channel `client_count` and `dmaengine_ref_count` are bare `long`s — overflow there silently demotes the "in use" guard, so the saturating refcount migration applies to both. The descriptor pool (per-driver `dma_pool`/`mempool`) must be `ZERO_ON_FREE` because completion descriptors carry buffer pointers and DMA addresses that may otherwise leak across reuse. Completion-callback dispatch is indirect through `tx->callback`/`tx->callback_result`/`tx->desc_free`; with W^X kernel text the callback table must be `__ro_after_init`-pinned for each registered driver.

`dma_request_chan` accepts an OF node from arbitrary devicetree — malformed `dmas` cells must be bounds-checked before driver-side `of_xlate` runs. Likewise ACPI `_DMA` is parsed without re-validation today; treat as untrusted.

Per-engine `device_prep_*` paths build SG lists from client pointers; the framework currently trusts clients to keep buffers DMA-mapped for transaction lifetime — buffer-unmap-before-completion is the most common abuse path and must be caught by Kani lifetime invariants.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `dma_device`, `dma_chan`, `dma_async_tx_descriptor`, `virt_dma_desc`, and per-driver descriptor pools so client copy_to/from_user paths cannot leak engine-internal pointers.
- **PAX_KERNEXEC** — `dma_device_ops`-equivalent function-pointer tables (per-engine `device_prep_dma_memcpy`, `device_tx_status`, `device_issue_pending`) live in W^X kernel text; per-driver init places them in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize stack across `dma_async_device_register`, `__dma_request_channel`, `vchan_complete` tasklet, and per-engine prep paths.
- **PAX_REFCOUNT** — saturating refcount on `dma_chan.client_count`, `dmaengine_ref_count`, and per-driver per-descriptor refcounts; overflow trap defeats request/release races that would UAF a freed engine.
- **PAX_MEMORY_SANITIZE** — zero-on-free on descriptor-pool slabs and per-channel HW ring caches so DMA-source/destination addresses and in-flight cookies do not bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN on every uapi entry that takes a client buffer pointer (dmatest, future iouring DMA helpers).
- **PAX_RAP / kCFI** — completion-callback dispatch (`tx->callback`, `tx->callback_result`, `tx->desc_free`) typed under kCFI; `device_prep_dma_*` vtable entries kCFI-typed.
- **GRKERNSEC_HIDESYM** — suppress `dma_chan`/`dma_device` pointer disclosure in `/sys/kernel/debug/dmaengine/summary` and per-channel sysfs unless CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict per-engine probe banners + dmatest result lines (which can fingerprint the platform) to CAP_SYSLOG readers.
- **Completion-callback RAP typing** — `dma_async_tx_callback` and `dma_async_tx_callback_result` are typed indirect-call sites; kCFI rejects any descriptor whose callback typesig drifts.
- **Descriptor-pool sanitize-on-free** — per-driver `dma_pool` freelists overwritten with the pool's poison pattern + zeroed at descriptor return; defense against late completion reading stale buffer pointers.
- **Channel-id allocator non-reusable across cycle** — `dma_ida` IDs aged before reuse so post-unregister stale references fault rather than alias a freshly-registered engine.
- **OF/ACPI `dmas`/`_DMA` cell bounds-checked** — refuse out-of-spec cell counts to defend devicetree-poisoning escalation paths.

Rationale: dmaengine is the kernel's hand-off boundary between client pointers and bus-master engines that can DMA arbitrary physical memory if mis-programmed. A refcount underflow on `client_count`, a callback-table corruption, or a descriptor-pool reuse leaking source addresses turns a benign offload into an arbitrary-read primitive. Saturating refcounts, kCFI on completion dispatch, zero-on-free pool freelists, and ID aging make the boundary structurally honest.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-engine internals (covered in `dmaengine-core.md` + per-driver Tier-3s)
- IDXD (covered in `idxd.md`)
- dma-mapping / dma-direct (covered under `kernel/dma/`)
- `dma-buf` (covered in `drivers/dma-buf/`)
- `async_tx` framework (covered under `crypto/async_tx/`)
- 32-bit-only paths
- Implementation code
