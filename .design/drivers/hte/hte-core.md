# Tier-3: drivers/hte/hte.c — HTE (Hardware TimeStamp Engine) framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/hte/hte.c
  - drivers/hte/hte-tegra194.c
  - include/linux/hte.h
-->

## Summary

HTE (Hardware TimeStamp Engine) is the kernel framework for SoC-integrated timestamp-capture peripherals — small fixed-function blocks that latch a free-running monotonic counter on the rising/falling edge of an external line (typically GPIO, occasionally an internal IRQ) and queue a (line_id, edge, monotonic_ts) event for consumer kernel subsystems. Today the only in-tree provider is Tegra194 (`hte-tegra194.c`); HTE is used primarily by GPIO descriptor consumers that need hardware-quality timestamps for line events without per-event ISR jitter and by perf/trace consumers needing sub-microsecond ground truth.

The framework is provider-callback symmetric: providers register a `hte_chip` exposing `request_ts_ns`, `release_ts_ns`, `enable_ts`, `disable_ts`, and `get_clk_src_info`; consumers obtain a `hte_ts_desc` via `hte_ts_get()` and a per-line callback (`hte_ts_cb_t`) that runs in workqueue context after the provider's IRQ drains a hardware FIFO. There is no character device interface — HTE is strictly an in-kernel framework — but every provider chip exposes a debugfs view under `/sys/kernel/debug/hte/<chip>/` for observability.

This Tier-3 covers `hte.c` (~940 lines: subsystem registration, provider/consumer matching by OF/DT phandle, per-descriptor request/enable/disable/release, debugfs root).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hte_chip` | per-provider device record + ops | `drivers::hte::Chip` |
| `struct hte_device` | per-provider HTE framework wrapper | `drivers::hte::Device` |
| `struct hte_ts_info` | per-line consumer-bound state | `drivers::hte::TsInfo` |
| `struct hte_ts_desc` | consumer-side descriptor handle | `drivers::hte::TsDesc` |
| `struct hte_ts_data` | per-event (raw_level, tsc, seq, ...) payload | `drivers::hte::TsData` |
| `hte_register_chip(chip)` / `hte_unregister_chip(chip)` | provider register/unregister | `Chip::register` / `_unregister` |
| `devm_hte_register_chip(parent, chip)` | devm-scoped provider register | `Chip::devm_register` |
| `hte_ts_get(dev, desc, index)` / `hte_ts_put(desc)` | consumer acquire / release | `TsDesc::get` / `_put` |
| `of_hte_req_count(dev)` | count consumer's `timestamps` OF phandles | `TsDesc::of_req_count` |
| `hte_request_ts_ns(desc, cb, tcb, data)` (`__hte_req_ts` plus wrapper macro) | bind callback + arm | `TsDesc::request` |
| `hte_enable_ts(desc)` / `hte_disable_ts(desc)` | runtime enable/disable | `TsDesc::enable` / `_disable` |
| `hte_push_ts_ns(chip, idx, &data)` | provider-side: deliver timestamp event | `Chip::push_ts_ns` |
| `hte_get_clk_src_info(desc, &cs)` | clock-source metadata for converters | `TsDesc::clk_src_info` |
| `hte_do_cb_work` | per-line workqueue trampoline | `TsInfo::do_cb_work` |
| `of_node_to_htedevice(np)` / `hte_find_dev_from_linedata(desc)` | provider lookup | `Device::lookup_*` |

## Compatibility contract

REQ-1: ABI `include/linux/hte.h` is kernel-internal (no UAPI); per-provider chip type, `hte_clk_src_info`, `hte_ts_data`, `hte_ts_desc.attr` layout must remain stable across kernel versions for in-tree providers and consumers.

REQ-2: Provider responsibilities — `hte_chip.ops.request_ts_ns(chip, line_id)` reserves a hardware FIFO/slot for the line; `enable_ts(chip, line_id)` arms the trigger; on hardware event the provider IRQ handler drains a FIFO and calls `hte_push_ts_ns(chip, line_id, &data)` per event.

REQ-3: Consumer responsibilities — derive a `hte_ts_desc` from devicetree (`timestamps` phandle list on the consumer node) or `data_name`-based lookup; obtain via `hte_ts_get`; arm with `hte_request_ts_ns(desc, cb, tcb, data)`; later `hte_disable_ts` + `hte_ts_put`.

REQ-4: Callback model — primary callback `hte_ts_cb_t cb(ts_data, data)` runs in workqueue (process) context after IRQ-side drain; optional threaded callback `hte_ts_sec_cb_t tcb` runs at IRQ-context first to enable low-latency consumers; both nullable.

REQ-5: Per-line uniqueness — at most one consumer descriptor bound to a given (chip, line_id) at any time; second `hte_request_ts_ns` for the same line returns `-EBUSY`.

REQ-6: Line-id space bounded by `hte_chip.nlines`; per-chip `ei[]` `__counted_by(nlines)` array enforced.

REQ-7: Clock-source semantics — `hte_clk_src_info { clk_freq, clk_resolution, raw_to_ns_factor }` published by provider via `get_clk_src_info`; consumers must convert raw `tsc` field to ns using these.

REQ-8: Devm symmetry — `devm_hte_register_chip` deregisters on parent device unbind; consumer descriptors bound to a chip that goes away receive `-ENODEV` on subsequent `enable_ts` and a wake on outstanding waiters.

REQ-9: debugfs — root `hte/` directory created subsys_initcall; per-chip directory + per-line subdirectory with `enabled`, `events`, and `dropped` files; debugfs writes are CAP_SYS_ADMIN-gated.

REQ-10: No userland ABI — HTE is in-kernel-only; userland access is via GPIO `linereq` (cross-ref `drivers/gpio/gpiolib-cdev.md`) when the GPIO consumer is HTE-backed.

## Acceptance Criteria

- [ ] AC-1: Tegra194 boot with `hte-tegra194` provider; provider visible at `/sys/kernel/debug/hte/<name>/`.
- [ ] AC-2: GPIO line-event consumer requests HTE timestamps via DT `timestamps = <&hte 0>` — receives wall-monotonic-coherent ts data on edge.
- [ ] AC-3: Two consumers race on the same line — second `hte_request_ts_ns` returns `-EBUSY`.
- [ ] AC-4: Provider unregister with consumer active — consumer's next `hte_enable_ts` returns `-ENODEV`; in-flight cb work drains.
- [ ] AC-5: `hte-tegra194-test` selftest runs to completion: timestamp delta vs. CLOCK_MONOTONIC within provider-declared resolution.
- [ ] AC-6: Disable + re-enable cycle preserves consumer state across one OFF window with no missed events while ENABLED.

## Architecture

`Device` lives in `drivers::hte::Device`:

```
struct Device {
  list: ListLinks,                 // global hte_devices list (hte_lock)
  chip: NonNull<Chip>,
  dbg_root: Option<DentryHandle>,
  refcount: Refcount,
  nlines: u32,
  ei: KBox<[TsInfo]>,              // __counted_by(nlines)
}

struct TsInfo {
  free: AtomicBool,
  flags: AtomicU64,                // HTE_TS_REGISTERED / _ENABLED / _DROPPED
  hte_cb_flags: u64,
  seq: AtomicU64,
  line_index: u32,
  cb_work: WorkStruct,
  cb: Option<HteTsCb>,
  tcb: Option<HteTsSecCb>,
  cb_data: *mut c_void,
  gdev: NonNull<Device>,
  desc: *mut TsDesc,
  dbg_root: Option<DentryHandle>,
}

struct Chip {
  dev: &Device,
  ops: &'static ChipOps,
  nlines: u32,
  data: *mut c_void,
  xlate_of: Option<XlateOf>,       // optional consumer-id xlate from OF args
  xlate_plat: Option<XlatePlat>,
  match_from_linedata: Option<MatchFn>,
  of_hte_n_cells: u32,
  hdev: *mut Device,
}
```

Consumer request `__hte_req_ts(desc, cb, tcb, data)`:
1. `hte_ts_get(dev, desc, index)` already resolved desc → ei via `hte_find_dev_from_linedata` or `hte_of_get_dev`.
2. Validate `ei.free == true`; CAS to `false` (returns `-EBUSY` on race).
3. Store `cb`, `tcb`, `data` into `ei`.
4. `INIT_WORK(&ei.cb_work, hte_do_cb_work)`.
5. `chip.ops.request_ts_ns(chip, line_index)` — provider reserves HW slot.
6. Provider returns; set `ei.flags |= REGISTERED`.
7. `hte_ts_dbgfs_init(name, ei)` installs per-line debugfs subdir.

Event delivery (provider side) `hte_push_ts_ns(chip, idx, data)`:
1. Look up `ei = chip.hdev.ei[idx]`.
2. If `ei.tcb` non-null: invoke from IRQ-context with `&data`.
3. Bump `ei.seq`; queue `ei.cb_work`.
4. Workqueue runs `hte_do_cb_work` → `ei.cb(data, ei.cb_data)`.

Enable/disable `hte_ts_dis_en_common(desc, en)`:
1. Resolve `ei = desc.hte_data`.
2. Acquire global `hte_lock`.
3. Call `chip.ops.enable_ts` or `_disable_ts(chip, ei.line_index)`.
4. Update `ei.flags` ENABLED bit.

Release `hte_ts_put(desc)`:
1. Mark `ei.flags &= ~REGISTERED`.
2. `chip.ops.release_ts_ns(chip, ei.line_index)` — provider frees HW slot.
3. `cancel_work_sync(&ei.cb_work)` to drain pending callbacks.
4. Clear `ei.cb`/`tcb`/`cb_data`, set `ei.free = true`, `desc.hte_data = NULL`.

Provider register `hte_register_chip(chip)`:
1. Allocate `hte_device` + `ei[nlines]` array.
2. Validate `chip.ops` (`request_ts_ns`, `release_ts_ns`, `enable_ts`, `disable_ts` non-null).
3. Add to global `hte_devices` list under `hte_lock`.
4. Install per-chip debugfs subdir.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `line_index_oob` | OOB | every per-line operation rejects `line_index >= chip.nlines`. |
| `ei_free_invariant` | RACE | CAS on `ei.free` ensures at most one consumer descriptor bound at a time. |
| `cb_work_no_uaf` | UAF | `cancel_work_sync` precedes consumer-data clear so workqueue cannot touch freed cb_data. |
| `device_list_uaf` | UAF | per-Device `refcount` held across consumer lookups so unregister waits for last consumer. |

### Layer 2: TLA+

`models/hte/event_delivery.tla` — proves IRQ-side `hte_push_ts_ns` enqueue + workqueue cb + concurrent `hte_disable_ts` + `hte_ts_put` cannot deliver a callback to a released `ei.cb_data`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `hte_ts_get` post: returns `Ok` only if `ei.free` was `true` before CAS | `TsDesc::get` |
| `hte_ts_put` post: provider's `release_ts_ns` invoked exactly once; `ei.free` restored to `true` | `TsDesc::put` |
| Provider `nlines` bound: every API call validates `line_index < nlines` | `Chip::*` |
| Per-event `seq` monotonic per `ei`: each push increments | `Chip::push_ts_ns` |

### Layer 4: Verus functional

`request_ts → enable_ts → N × push_ts_ns → disable_ts → release_ts` schedule encodes a regular sequence with no event-loss while ENABLED and exactly the events between disable and re-enable dropped. Verus encoding proves the cb count equals the number of pushes that occurred in ENABLED windows.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

hte-core specific reinforcement:

- **Per-chip `ei[]` `__counted_by`** — provider `nlines` bounds every line-index access; defense against off-by-one in xlate.
- **Global `hte_lock` serializes register/unregister + consumer lookup** — defense against per-list racing during chip teardown.
- **Devm-bound provider lifetime** — parent device unbind frees the chip safely; consumers' next op observes `-ENODEV`.
- **cancel_work_sync on put** — defense against workqueue cb firing on freed consumer data.
- **debugfs entries CAP_SYS_ADMIN-only** — debugfs root historically owns this; explicit per-file perm check on writable nodes.
- **Per-line callback work bounded** — single per-line workqueue work item, no unbounded enqueue (driver replaces queued work on overflow).

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `hte_device`, `hte_ts_info`, and the per-line callback `data` pointers; debugfs `events`/`dropped` exports size-validated.
- **PAX_KERNEXEC** — HTE core text + per-provider `hte_ops` op-vectors in W^X kernel text; `__ro_after_init` for the global `hte_devices` list head and debugfs root dentry.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `hte_ts_get`, `__hte_req_ts`, `hte_ts_put`, `hte_push_ts_ns`, and per-provider IRQ handlers feeding `hte_push_ts_ns`.
- **PAX_REFCOUNT** — saturating `refcount_t` on `hte_device` and per-line `hte_ts_info` references; overflow trap defeats consumer-vs-provider unregister race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-line `ei` slots on provider unregister, callback `data` pointers cleared on `hte_ts_put`, and debugfs ring buffers so stale timestamps cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every debugfs write into HTE (per-line enable/disable, drop-count clear); no user-pointer paths via UAPI.
- **PAX_RAP / kCFI** — `hte_chip.ops` (`request_ts_ns`, `release_ts_ns`, `enable_ts`, `disable_ts`, `get_clk_src_info`), per-consumer `cb` / `tcb`, and `xlate_of`/`xlate_plat` callbacks marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-provider MMIO pointer disclosure behind CAP_SYSLOG; suppress `%p` in HTE-provider registration banners and event tracepoints.
- **GRKERNSEC_DMESG** — restrict provider-register, line-overflow-dropped, and consumer-unregister banners to CAP_SYSLOG so attackers cannot fingerprint timestamp consumers via dmesg.
- **TS event PAX_USERCOPY** — timestamp event payloads exported via debugfs trace ring sized and bounded; never copy raw provider-pointer fields to userland.
- **Line-id bounds** — every API call validates `line_id < chip.nlines` at the framework layer; defense against provider xlate trust.
- **`hte_request_ts_ns` CAP_SYS_ADMIN** — kernel callers are in-tree but consumer-bridging interfaces (GPIO HTE-on-line cdev) require CAP_SYS_ADMIN at the bridge layer.
- **Callback RAP** — per-consumer `cb` / `tcb` invoked through kCFI-typed call sites; refuse cb registration without correct type id.
- **debugfs CAP_SYS_ADMIN** — `/sys/kernel/debug/hte/` writable entries gated by CAP_SYS_ADMIN; refuse mount-time access in locked-down kernels.
- **Workqueue overflow bound** — per-line cb_work is single-shot; provider replaces queued work atomically on overflow so a stuck consumer cannot wedge kernel workqueue.

Rationale: HTE timestamps feed performance counters, latency telemetry, and gpio line-event handlers that are often security-relevant (door-sensor, debounce, key-event capture). A consumer/provider lifetime race or callback type confusion can be turned into a UAF or function-pointer hijack. RAP/kCFI on chip ops + consumer callbacks, refcount-overflow trapping, sanitised release, and CAP_SYS_ADMIN gating of bridge UAPI turn HTE into a structurally-safe kernel-internal framework rather than an undisciplined callback registry.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Tegra194 provider details (covered in `hte-tegra194.md` future Tier-3).
- GPIO consumer bridge (covered in `gpiolib-cdev.md`'s HTE section).
- Per-event ring buffer details (covered in `hte-debugfs.md` future Tier-3).
- 32-bit-only paths
- Implementation code
