# Tier-3: drivers/hwtracing/coresight/* — ARM CoreSight framework (sources, links, sinks, ETMv4, ETB/ETR, perf integration)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/hwtracing/coresight/coresight-core.c
  - drivers/hwtracing/coresight/coresight-etm-perf.c
  - drivers/hwtracing/coresight/coresight-etm4x-core.c
  - drivers/hwtracing/coresight/coresight-tmc-core.c
  - drivers/hwtracing/coresight/coresight-tmc-etr.c
  - drivers/hwtracing/coresight/coresight-tmc-etf.c
  - drivers/hwtracing/coresight/coresight-funnel.c
  - drivers/hwtracing/coresight/coresight-replicator.c
  - drivers/hwtracing/coresight/coresight-sysfs.c
  - drivers/hwtracing/coresight/coresight-syscfg.c
  - drivers/hwtracing/coresight/coresight-platform.c
  - include/linux/coresight.h
-->

## Summary

ARM CoreSight is an on-die tracing fabric standardized by Arm — every modern ARMv8/ARMv9 SoC ships some subset of CoreSight components: per-core Embedded Trace Macrocell (ETMv4 / ETMv4.x / ETE), funnels, replicators, Trace Memory Controllers (TMC) in ETB (Embedded Trace Buffer), ETF (Embedded Trace FIFO), or ETR (Embedded Trace Router to system memory) modes, plus optional STM (System Trace Macrocell), CTI (Cross Trigger Interface), CATU (Coresight Address Translation Unit), and per-vendor TPDM/TPDA aggregators.

The CoreSight framework provides: a `coresight_bustype` device class binding amba/platform devices to a typed `coresight_device` (source / link / sink / helper / panic), a per-path graph walker that turns "trace source X → sink Y" into a chain of enable/disable calls, sysfs control (`/sys/bus/coresight/devices/<dev>`), perf integration via the `cs_etm` PMU (one PMU multiplexed across per-cpu ETM sources writing into a chosen sink), kernel configfs trace-configurations (`coresight-syscfg`), and a panic-trace path so post-mortem trace can be captured into ETR-backed RAM.

This Tier-3 covers `coresight-core.c` (~1808 lines: bus, register/unregister, path build, mode arbitration), `coresight-etm-perf.c` (~1017 lines: perf AUX integration), `coresight-etm4x-core.c` (~2641 lines: ETMv4 source), and `coresight-tmc-*` (~3957 lines: ETB/ETF/ETR sink).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct coresight_device` | per-device control block (source/link/sink) | `drivers::coresight::CsDevice` |
| `struct coresight_ops` (`ops_sink`, `ops_link`, `ops_source`, `ops_helper`, `ops_panic`) | per-type vtable | `drivers::coresight::CsOps` |
| `struct coresight_platform_data` | per-device connections + in/out ports parsed from OF/ACPI | `drivers::coresight::PlatformData` |
| `struct coresight_path` | resolved source→sink list per perf event | `drivers::coresight::Path` |
| `struct coresight_trace_id_map` | per-sink trace-id allocator | `drivers::coresight::TraceIdMap` |
| `coresight_register(desc)` / `coresight_unregister(csdev)` | bus add/remove | `CsDevice::register` / `_unregister` |
| `coresight_enable_sysfs(csdev)` / `_disable_sysfs(csdev)` | sysfs path enable | `CsDevice::enable_sysfs` / `_disable_sysfs` |
| `coresight_take_mode(csdev, mode)` / `coresight_set_mode(...)` / `coresight_get_mode(...)` | mode arbitration (sysfs vs perf vs panic) | `CsDevice::take_mode` |
| `coresight_claim_device(csdev)` / `_disclaim_device(...)` | per-CLAIM-register HW-arbitration of external debugger vs kernel | `CsDevice::claim_device` |
| `etm_event_init` / `etm_event_add` / `etm_event_del` / `etm_event_start` / `etm_event_stop` | perf PMU callbacks for `cs_etm` | `EtmPmu::*` |
| `etm_setup_aux(event, pages, nr, overwrite)` / `etm_free_aux(data)` | perf-AUX buffer alloc | `EtmPmu::setup_aux` / `_free_aux` |
| `tmc_register(adev, tmc, csdev)` / `tmc_enable_etf_sink(...)` / `_enable_etr_sink(...)` / `_disable_etr_sink(...)` | TMC sink driver | `Tmc::enable_etf_sink` / `_etr_sink` |
| `tmc_alloc_etr_buffer(csdev, event, pages, nr_pages, snapshot)` | per-event ETR sg-buffer alloc | `Tmc::alloc_etr_buffer` |
| `etm4_enable_perf(csdev, event)` / `_disable_perf(csdev, event)` / `_enable_sysfs(...)` / `_disable_sysfs(...)` | ETMv4 source enable/disable | `Etm4::enable_perf` / `_sysfs` |
| `cscfg_load_features` / `cscfg_activate_config(...)` | configfs-driven trace-configuration apply | `Syscfg::activate_config` |
| `coresight_register_panic_notifier()` / `coresight_panic_sync()` | panic-time trace drain | `Panic::drain` |

## Compatibility contract

REQ-1: CoreSight bus (`coresight_bustype`) groups devices by `enum coresight_dev_type` (SOURCE / LINK / SINK / LINKSINK / HELPER / ECT). Each device exposes typed `coresight_ops` validated at `coresight_register()`.

REQ-2: Per-device connections discovered from OF (`coresight-platform.c`) or ACPI graph properties; per-port `in_conns` / `out_conns` populated; orphan-connection list resolved as devices appear.

REQ-3: Per-path build from a source to a sink uses BFS over connections; per-link `link_ops->enable(csdev, inport, outport)` invoked in order; sink-enable last; disable in reverse.

REQ-4: Mode arbitration: a source / sink may be in `CS_MODE_DISABLED`, `CS_MODE_SYSFS`, `CS_MODE_PERF`, or `CS_MODE_PANIC` at any instant; `coresight_take_mode()` is atomic test-and-set; sysfs cannot preempt perf and vice versa.

REQ-5: Trace IDs allocated per-sink from `coresight_trace_id_map` bitmap (per-AMBA-spec 7-bit ID, reserved values excluded); per-source ID stable across enable cycles when the sink is unchanged.

REQ-6: TMC sink modes: ETB (on-die SRAM circular buffer), ETF (on-die FIFO, also usable as link), ETR (DMA into system RAM via scatter-gather list); per-mode buffer-alloc paths.

REQ-7: ETR scatter-gather: per-page DMA-coherent buffer pages strung into TMC SG-table; per-page descriptor written into the TMC `RWP` advancing register; wrap-around in circular mode.

REQ-8: ETMv4 (`coresight-etm4x-core.c`): per-CPU instance; per-CPU PMU "cs_etm" registered (one PMU multiplexing per-CPU sources); `etm_event_add` selects sink (via attribute `sinks=`), builds path, allocates AUX buffer, programs ETMv4 trace-config registers from `etmv4_config`.

REQ-9: perf AUX buffer interaction: per-event `etm_setup_aux` allocates `etm_event_data` containing per-CPU path + sink-specific buffer; `etm_event_start` enables the path; `etm_event_stop` disables + drains.

REQ-10: configfs trace configurations: `/sys/kernel/config/cs-syscfg/` lets userland load `cscfg_config_desc` + `cscfg_feature_desc` defining ETM register presets activatable per-event.

REQ-11: Panic notifier: registered devices with `ops_panic->sync(csdev)` flush trace state into ETR/ETB so post-mortem readers can recover the last N instructions.

REQ-12: ACPI / OF platform-data parsing rejects malformed connections (out-of-range port, dangling phandle); orphan connections retained until matching device probes.

## Acceptance Criteria

- [ ] AC-1: On a CoreSight-equipped ARM dev board (e.g. Juno R2, RK3399, Snapdragon 8cx), `ls /sys/bus/coresight/devices/` enumerates funnels, replicators, TMC, ETM per cluster.
- [ ] AC-2: sysfs single-CPU trace: `echo 1 > tmc_etf0/enable_sink && echo 1 > etm0/enable_source` produces a trace dump readable via `tmc_etf0` mmap.
- [ ] AC-3: `perf record -e cs_etm/@tmc_etr0/ --per-thread -- ./victim` captures decodable ETMv4 trace; `perf script` via OpenCSD decodes branches.
- [ ] AC-4: Two perf sessions on disjoint CPUs sharing a sink fail the second cleanly with `-EBUSY`; sharing across disjoint sinks succeeds.
- [ ] AC-5: configfs preset (`coresight-cfg-afdo`) activated produces an ETM trace with AutoFDO-friendly cycle-count + timestamp packets.
- [ ] AC-6: Panic-trace integration: forced panic → ETR buffer drained → bootloader-preserved memory contains the tail of the trace.
- [ ] AC-7: KUnit `coresight-kunit-tests.c` passes.
- [ ] AC-8: Hot-unplug a CPU during perf record → its ETM source quiesces without leaking trace IDs.

## Architecture

`CsDevice` core layout:

```
struct CsDevice {
  dev_type: CoresightDevType,   // SOURCE / LINK / SINK / LINKSINK / HELPER / ECT
  subtype: CoresightDevSubtype, // PROC / BUS / SOFTWARE / MEM / TPIU / SWO / PORT_LINK / etc.
  ops: &'static CsOps,
  csa: CsdevAccess,             // io_base + read/write callbacks
  refcnt: AtomicI32,            // for enable accounting
  mode: AtomicEnum<CsMode>,
  pdata: KBox<PlatformData>,
  trace_id_map: Option<Arc<TraceIdMap>>,
  enabled_sink: bool,
  activated: bool,              // sysfs-activated sink hint
  ea: Option<KBox<DevExtAttribute>>,
  features_csdev_list: Mutex<Vec<Arc<CscfgFeatureCsdev>>>,
  config_csdev_list: Mutex<Vec<Arc<CscfgConfigCsdev>>>,
}

struct Path {
  path_list: Vec<Arc<CsDevice>>, // sink-first iteration order
  trace_id: u8,
  sink_buffer: Option<KBox<SinkBuffer>>,
}
```

Probe + register flow `CsDevice::register`:
1. Driver (etm4x, tmc, funnel, replicator, stm, cti) probes its AMBA / platform device.
2. Build `coresight_desc { type, subtype, ops, pdata, dev }`.
3. `coresight_register(desc)` allocates `coresight_device`, runs `device_register()` against `coresight_bustype`, walks `pdata.out_conns` + matches into already-registered devices, links the device into `coresight_orphan_conns` if any port is unresolved.
4. Re-walk orphans on every new register to fix late-arriving connections.

Per-path build `coresight_build_path(source, sink, sink_buffer)`:
1. BFS from `source` through `out_conns` until `sink` is reached.
2. Each visited link/sink appended to `Path::path_list` (sink-first).
3. Each device's mode taken via `coresight_take_mode(csdev, mode)`; on failure unwind acquired modes.
4. Trace ID allocated from sink-owned map.

Enable order `coresight_enable_path(path)`:
1. Sink enable: `ops_sink->enable(csdev, sink_mode, sink_buffer)`.
2. Each intermediate link (reverse of BFS) `ops_link->enable(csdev, inport, outport)`.
3. Source enable last: `ops_source->enable(csdev, perf_event, mode)`.
4. On any failure, disable already-enabled prefix in reverse.

ETMv4 source enable (`etm4_enable_perf`):
1. Load per-cpu `etmv4_config` derived from event attrs + configfs configurations.
2. Save CPU's current trace-state, claim the unit via CLAIM register.
3. Program TRCCONFIGR, TRCEVENTCTL{0,1}R, TRCSTALLCTLR, TRCSYNCPR, TRCCCCTLR, TRCBBCTLR, TRCTRACEIDR, address-comparator + counter banks per config.
4. Set TRCPRGCTLR.EN; spin on TRCSTATR.IDLE clear with `coresight_timeout`.

TMC ETR sink enable (`tmc_enable_etr_sink_perf`):
1. Validate the chosen ETR buffer matches `event->cpu` affinity and the negotiated SG table.
2. Program AXICTL, FFCR, RSZ; write RWP / RRP / DBALO / DBAHI from SG-table descriptor list.
3. Set CTL.TRACECAPTEN, wait FFSR.READY.
4. On stop: assert manual flush, wait FFSR.FtEmpty, copy RWP into per-event buffer state, snapshot RAM pages into the perf AUX area.

Perf integration `etm_event_add`:
1. Resolve `event->attr.config2` "sinks=" attribute to `coresight_device *sink`.
2. `etm_setup_aux(event, pages, nr_pages, overwrite)` builds `Path` per cpu in event mask.
3. `etm_event_start(event, flags)` enables path for the event's cpu.
4. AUX-output managed via `perf_aux_output_begin` / `_end` with TMC residual byte count.

Mode arbitration: `coresight_take_mode` uses `cmpxchg(&csdev->mode, CS_MODE_DISABLED, requested)` so sysfs/perf/panic never silently collide; per-device refcount tracks "how many active paths use me as link" so a shared funnel does not disable while another source uses it.

## Hardening

- **Per-source CLAIM-register arbitration** — every source/sink/link does HW CLAIM_CLR/SET before MMIO programming so a JTAG debugger does not race kernel programming.
- **`coresight_timeout` bounded polling** — every register-bit wait carries an iteration cap; defense against hung MMIO causing CPU stalls.
- **Trace-ID allocation bitmap** — defense against duplicate-trace-id collisions corrupting decoded traces.
- **Per-path mode atomic test-and-set** — `cmpxchg`-based mode acquisition; defense against double-enable races.
- **Sink buffer page-locked + DMA-mapped under per-event ownership** — defense against ETR DMA into freed pages on event teardown.
- **Hot-unplug source quiesce** — per-CPU ETM CPU-PM notifier suspends trace before CPU power-down; restore on power-up.
- **CONFIG_CORESIGHT_SOURCE_ETM4X gated** — driver only loads on ETMv4-present CPUs (TRCDEVARCH check); refuse if device-tree advertises ETMv4 but TRCDEVARCH disagrees.
- **Panic-handler bounded** — panic notifier walks devices with `ops_panic->sync`; per-callout time-capped to avoid blocking panic.
- **Orphan-connection list bounded** — out-of-range port indices rejected at `coresight_get_platform_data` parse; orphan list never grows past device count.
- **CTI cross-trigger gated CAP_SYS_ADMIN** — CTI mux programming is a debug primitive; restricted.
- **Trace capture into kernel RAM via ETR refuses pages overlapping kernel image** — defense against trace overwriting `.text`.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `coresight_device`, `coresight_path`, `etm_event_data`, `etmv4_drvdata`, `tmc_drvdata`, and ETR SG-table descriptors; reject userspace copy of register dumps outside fixed-size sysfs buffers.
- **PAX_KERNEXEC** — CoreSight core (`coresight-core.c`) and per-type drivers placed in W^X kernel text; `coresight_ops` (`ops_sink`/`ops_link`/`ops_source`/`ops_helper`/`ops_panic`) marked `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `etm_event_add`, `etm_event_start`, `coresight_enable_sysfs`, `tmc_enable_etr_sink`, and panic-notifier entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `coresight_device.refcnt`, `coresight_path` path-list reference, ETR SG-table refcount, and per-PMU event reference so enable/disable storms cannot underflow.
- **PAX_MEMORY_SANITIZE** — zero-on-free for ETR SG buffers, ETB internal SRAM mirror, `etmv4_config` register caches, and TMC residual-byte state so a previous tenant's trace cannot leak into a fresh capture.
- **PAX_UDEREF** — SMAP/PAN enforced on sysfs and configfs entry; reject user-pointer deref outside canonical `kobj_attribute` show/store helpers.
- **PAX_RAP / kCFI** — `coresight_ops`, ETM PMU `pmu_ops`, TMC sink/buffer-ops, and CTI helper-ops marked `__ro_after_init` and dispatched via kCFI-typed indirect calls.
- **GRKERNSEC_HIDESYM** — gate CoreSight register-base + per-csdev pointer disclosure in `/proc/iomem`, debugfs, and tracepoint output behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict CoreSight enable/disable banners and "trace overflow" messages to CAP_SYSLOG; attackers must not be able to side-channel trace activity via dmesg.
- **CAP_SYS_RAWIO on sysfs trace control** — `enable_source`, `enable_sink`, `mgmt/*` writable attributes require CAP_SYS_RAWIO so unprivileged users cannot start CPU-tracing of other tenants.
- **CAP_SYS_ADMIN on configfs trace-configurations** — `cscfg_activate_config` and per-feature parameter writes require CAP_SYS_ADMIN since they program ETM internals.
- **Sink mmap NX-bit forced** — `/sys/.../sink/buffer` mmap returns PROT_READ|PROT_WRITE only, never PROT_EXEC, so a captured trace page cannot be turned into shellcode.
- **ETMv4 config validated against TRCIDR0..N feature register** — refuse to enable trace features the HW does not advertise (e.g. branch-broadcast on a unit that lacks BB support).
- **Trace ringbuf PAX_USERCOPY-bounded** — perf AUX copy paths use bounded `copy_to_user` helpers; OOB sink overrun rejects the event.
- **HIDESYM in trace** — symbol annotation in ETM trace decode (`perf script`) gated so unprivileged users see address-only output.

Rationale: CoreSight is a kernel-resident JTAG. A successful enable as an unprivileged user gives an attacker a CPU-instruction-level side-channel into other tenants (KASLR defeat, AES key recovery, branch-history exfil). Every primitive — sysfs enable, perf "cs_etm" PMU, configfs preset, panic drain — must gate on the appropriate capability, refuse mode-races, and keep ETR DMA inside the per-event buffer. RAP/kCFI on `coresight_ops` prevents the natural escalation path of "swap a sink op pointer to RWX gadget on an ETR write."

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-vendor TPDM/TPDA aggregators (covered in `coresight-tpdm.md` future Tier-3)
- STM (System Trace Macrocell) details (covered in `coresight-stm.md` future Tier-3)
- ETMv3 / ETM3x (legacy, covered separately)
- CATU translation specifics (covered in `coresight-catu.md` future Tier-3)
- DFL/AFU OPAE FPGA management (orthogonal subsystem)
- 32-bit-only paths
- Implementation code
