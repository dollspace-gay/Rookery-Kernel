# Tier-3: drivers/memory/{emif,fsl_ifc,mtk-smi,tegra,...} — SoC memory-controller framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/memory/emif.c
  - drivers/memory/emif.h
  - drivers/memory/fsl_ifc.c
  - drivers/memory/fsl-corenet-cf.c
  - drivers/memory/mtk-smi.c
  - drivers/memory/tegra/mc.c
  - drivers/memory/tegra/tegra20-emc.c
  - drivers/memory/tegra/tegra30-emc.c
  - drivers/memory/tegra/tegra124-emc.c
  - drivers/memory/tegra/tegra210-emc-core.c
  - drivers/memory/tegra/tegra186-emc.c
  - drivers/memory/atmel-ebi.c
  - drivers/memory/jz4780-nemc.c
  - drivers/memory/omap-gpmc.c
  - drivers/memory/pl172.c
  - drivers/memory/pl353-smc.c
  - drivers/memory/samsung/exynos5422-dmc.c
  - drivers/memory/ti-emif-pm.c
  - drivers/memory/of_memory.c
-->

## Summary

`drivers/memory/` is the collection of SoC memory-controller drivers — there is *no* single "memory bus" framework analogous to `parport_bus_type`; instead each SoC's external memory interface (EMIF on TI Keystone/AM33xx/AM43xx/AM57xx, IFC on Freescale/NXP QorIQ, GPMC on TI OMAP, EBI on Atmel, FMC2 on STM32, NEMC on Ingenic, Devbus on Marvell, MC/EMC on NVIDIA Tegra, DMC on Samsung Exynos, SMI on MediaTek, SROM on Exynos, PL172/PL353 on ARM PrimeCell) registers as a platform_device or interconnect provider. The shared patterns are: parse DT/ACPI memory timings via `of_memory.c` JEDEC helpers, drive frequency-scaling notifiers (`devfreq` or PM `qos`), expose ECC error injection / report tracepoints, manage NAND/NOR/SRAM chip-select windows on multi-chip-select EBI-style controllers, and on systems with interconnect-tracked bandwidth (Tegra, Mediatek SMI, Exynos DMC) participate in the `drivers/interconnect/` framework.

This Tier-3 documents the *cross-driver patterns* — the EMIF (TI) and Tegra MC/EMC families as the two canonical references (LPDDR2/3/4 timing scaling + thermal de-rating + ECC) plus the MediaTek SMI multi-larb arbiter that bridges the memory controller and the IOMMU. Per-SoC specifics live in Tier-4 docs.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct emif_data` (`drivers/memory/emif.c`) | per-EMIF instance state (timings, IRQ, devfreq notif) | `drivers::memory::emif::Emif` |
| `struct lpddr2_timings` / `_min_tck` / `_addressing` / `_device_info` (`include/memory/jedec_ddr.h`) | JEDEC LPDDR2/3 DRAM characterization | `drivers::memory::JedecLpddr*` |
| `of_lpddr2_get_info(np, dev)` / `of_get_min_tck` / `of_get_ddr_timings` (`of_memory.c`) | DT JEDEC-DDR parsers | `drivers::memory::OfMemory::*` |
| `struct tegra_mc` (`drivers/memory/tegra/mc.h`) | per-Tegra-SoC memory-controller (MC) state | `drivers::memory::tegra::Mc` |
| `struct tegra_mc_client` | per-MC client (display, GPU, ISP, MSENC, etc.) with priority + smmu_id | `drivers::memory::tegra::McClient` |
| `tegra_mc_probe_device(mc, dev)` / `tegra_mc_write_emem_configuration(mc, rate)` / `tegra_mc_get_emem_device_count(mc)` / `tegra_mc_get_carveout_info(mc, id, base, size)` / `devm_tegra_memory_controller_get(dev)` | Tegra MC public API | `tegra::Mc::*` |
| `struct mtk_smi_larb` / `mtk_smi_common_plat` / `mtk_smi_larb_gen` (`drivers/memory/mtk-smi.c`) | Mediatek SMI local arbiter / common arbiter | `drivers::memory::mtk::SmiLarb` / `SmiCommon` |
| `mtk_smi_device_link_common(dev)` / `mtk_smi_larb_get(dev)` / `_put(dev)` | per-larb power-domain link helper | `mtk::SmiLarb::*` |
| `struct fsl_ifc_global` / `fsl_ifc_runtime` (`drivers/memory/fsl_ifc.c`) | Freescale Integrated Flash Controller global + per-bank | `drivers::memory::fsl_ifc::Ifc` |
| `struct atmel_ebi` / `atmel_ebi_dev` (`drivers/memory/atmel-ebi.c`) | Atmel External Bus Interface per-CS | `drivers::memory::atmel_ebi::Ebi` |
| `struct gpmc_settings` (`include/linux/platform_data/gpmc-omap.h`) | TI OMAP General Purpose Memory Controller per-CS | `drivers::memory::omap_gpmc::*` |
| `struct exynos5_dmc` (`drivers/memory/samsung/exynos5422-dmc.c`) | Exynos5422 DMC with devfreq | `drivers::memory::samsung::Dmc5422` |
| `ti_emif_save_context()` / `ti_emif_restore_context()` / `ti_emif_run_hw_leveling()` (`drivers/memory/ti-emif-pm.c`) | EMIF suspend/resume hooks called from secure-PM | `drivers::memory::ti_emif::pm::*` |
| `emif_calculate_regs(emif, freq, &regs)` / `setup_temperature_sensitive_regs(emif, regs)` / `volt_notify_handling(emif, &nb_info)` | EMIF timing + thermal + DVFS callbacks | `emif::Emif::*` |

## Compatibility contract

REQ-1: Each memory-controller driver registers as a `platform_driver` against DT `compatible` strings; no shared bus_type — drivers are bound 1:1 to a SoC controller block.

REQ-2: DT timing parse: every JEDEC-DDR-aware driver uses `of_memory.c` helpers — `of_lpddr2_get_info()`, `of_get_min_tck()`, `of_get_ddr_timings()` — which return canonical `struct lpddr2_*` values from `include/memory/jedec_ddr.h`.

REQ-3: EMIF (TI) supports DDR2 / LPDDR2 / DDR3 / LPDDR3 with HW-leveling and per-temperature de-rating; per-EMIF IRQ services thermal events + ECC error reports.

REQ-4: Tegra MC arbitrates per-client bandwidth via per-client priority + SMMU stream-id; per-Tegra-generation EMC (External Memory Controller) provides DRAM-frequency scaling under `devfreq` / `cpufreq` / interconnect QoS — the MC and EMC are paired but separately bound.

REQ-5: Mediatek SMI implements a two-tier arbiter (per-IP "larb" + global "common") that sits between the IOMMU and the DRAM controller; each larb's clock/power-domain is reference-counted via `mtk_smi_larb_get/put`.

REQ-6: Freescale IFC (Integrated Flash Controller) is multi-purpose: NAND, NOR, SRAM, GPCM (generic), CPLD; per-bank `ifc_bank` config picks function + timing; one global IRQ services per-bank events.

REQ-7: Atmel EBI / TI GPMC / Ingenic NEMC / Marvell Devbus / ARM PrimeCell PL172/PL353 / STM32 FMC2 are all multi-chip-select static-memory controllers — per-CS DT child node configures `chip-select`, `bank-width`, timing parameters, and (for PL353/GPMC/NEMC) a per-CS child platform device for the attached chip (NAND flash, FPGA bridge, etc.).

REQ-8: Exynos5422 DMC drives devfreq with hardware performance counters (PPMU) and EW (Early Warning) ECC interrupts; per-event handler updates per-frequency timing registers.

REQ-9: PM suspend/resume: every memory-controller driver implements `pm_ops.suspend`/`.resume` (some via PSCI / secure-monitor on ARM); `ti-emif-pm.c` exposes asm-coded `ti_emif_sram_pm` for early-resume DRAM-self-refresh exit (cannot use C runtime).

REQ-10: ECC reporting: per-controller integrates with `drivers/edac/` where applicable (cross-ref `drivers/edac/`) — for EMIF/IFC/DMC, the EDAC layer registers tracepoints and per-DIMM error counters.

REQ-11: Interconnect framework: Tegra (`tegra_mc_icc_xlate`) and Mediatek SMI register as `drivers/interconnect/` providers, allowing clients to express bandwidth requirements via `of_icc_get`.

REQ-12: Memory carveouts: Tegra MC exposes per-client carveout regions via `tegra_mc_get_carveout_info()` — pre-reserved physical ranges for GPU/display/codec, established at bootloader handoff.

## Acceptance Criteria

- [ ] AC-1: On a TI AM57xx board, `modprobe ti-emif-sram` then suspend/resume cycles preserve DRAM contents.
- [ ] AC-2: On Tegra X1, `cat /sys/kernel/debug/tegra-mc/clients` enumerates display / GPU / VIC clients with priority + SMMU stream-id.
- [ ] AC-3: Tegra X1 EMC frequency scaling: write a higher freq via devfreq and observe `tegra_mc_write_emem_configuration` reprogram timing without bus stall.
- [ ] AC-4: MT8195 SMI: `mtk_smi_larb_get` from display driver brings larb clock on; matching `_put` drops it; refcount visible via debugfs.
- [ ] AC-5: Freescale T1024 IFC: NAND attached on CS0 enumerates via `nand_scan_ident`; per-CS timing registers programmed from DT.
- [ ] AC-6: ECC injection on EMIF (CS-1bit toggle): EDAC reports CE; uncorrectable double-bit causes UE log without crashing.
- [ ] AC-7: Exynos5422 DMC: devfreq `userspace` governor sets DRAM freq; PPMU counter increments observed.
- [ ] AC-8: All in-tree memory-controller drivers build clean under `allmodconfig` for their respective arch.

## Architecture

Per-driver pattern (taking EMIF as canonical):

```
struct Emif {
  base: NonNull<u8>,                 // MMIO of EMIF registers
  ip_rev: u32,                       // EMIF1/EMIF2/EMIF4D/EMIF4D5
  phy_type: PhyType,                 // attaching DDR PHY family
  dev_info: Arc<LpddrInfo>,
  plat_data: Arc<EmifPlatformData>,
  min_tck: Arc<LpddrMinTck>,
  timings: Vec<LpddrTimings>,        // per-supported-frequency
  duration_in_ns_*: u32,             // various calibration constants
  freq_table: Vec<u32>,
  curr_regs: AtomicPtr<EmifRegs>,    // active timing reg set
  regs_cache: Mutex<Vec<Arc<EmifRegs>>>,
  irq: u32,
  temperature_level: AtomicU32,      // SDRAM_TEMP_NOMINAL/HIGH/SHRINK
  lpmode: AtomicU32,                 // low-power mode selection
  cpufreq_notif: Notifier,
  thermal_notif: Notifier,
}

struct Mc {                          // Tegra MC
  soc: &'static TegraMcSoc,          // per-generation desc (clients, smmu groups, etc.)
  regs: NonNull<u8>,
  ch_regs: Vec<NonNull<u8>>,          // per-channel MC (T18x+ has multi-channel)
  bcast_regs: NonNull<u8>,
  reset: Reset,
  clk: Clk,
  irq: u32,
  smmu: Option<Arc<TegraSmmu>>,       // owns the iommu_ops (cross-ref drivers/iommu/tegra-smmu)
  icc_provider: IccProvider,          // drivers/interconnect provider
  channels: Vec<TegraMcChannel>,
  err_mask: u32,                      // for ::interrupt mask
  ops: &'static TegraMcOps,           // per-SoC vtable
}

struct SmiLarb {                     // MediaTek
  smi: KBox<Smi>,                    // shared base region
  larb_gen: &'static SmiLarbGen,     // per-SoC desc (port count, mmu_en mask)
  larb_id: u32,
  smi_common_dev: Arc<Device>,
  mmu: AtomicU32,                    // per-port enable mask
  larb_pwr_consumer: AtomicU32,      // ref count
  ostdl: Vec<u32>,                   // outstanding port limits
}
```

Per-driver lifecycle (EMIF as canonical):
1. `platform_probe(pdev)`: parse DT for `ti,hwmods` + `ddr_device_info` child + `min-tck` child + per-frequency `timings`.
2. `of_get_ddr_timings` / `_get_min_tck` / `_lpddr2_get_info` populate the structures from JEDEC parsers.
3. `emif_calculate_regs(emif, freq, &regs)` precomputes per-frequency timing reg blob; cached in `regs_cache`.
4. `request_irq(emif.irq, emif_interrupt_handler, IRQF_SHARED, emif.dev_name, emif)`.
5. Register CPUFreq + thermal notifiers so DVFS and temperature trigger `setup_temperature_sensitive_regs` rewrites.
6. Expose debugfs nodes (`/sys/kernel/debug/emif/`) for active timings + regs_cache.

DVFS path (EMIF, on CPUFreq transition):
1. `volt_notify_handling(emif, &nb_info)`: target voltage delta → look up which `EmifRegs` set matches the new freq.
2. `setup_registers(emif, regs)` writes new timing reg blob in correct sequence (precharge-all + tDLL-relock).
3. CAS `emif.curr_regs` to new `Arc<EmifRegs>`.

Thermal path (EMIF, on `EMIF_IRQ_THERMAL_RISING` IRQ):
1. Read `SDRAM_THERMAL_LEVEL` from EMIF status.
2. Update `emif.temperature_level`.
3. `setup_temperature_sensitive_regs(emif, regs)` rewrites refresh interval (tREFI/tREFI/tREFIabank scaled per thermal level).
4. Re-enable IRQ.

Tegra MC client probe (per-IP):
1. Display/GPU/VIC/etc. driver calls `devm_tegra_memory_controller_get(dev)` returning the SoC's `Arc<Mc>`.
2. `tegra_mc_probe_device(mc, dev)` finds matching `TegraMcClient` entry by stream-id, links the device into `mc.dev_list`.
3. SMMU stream-id auto-bound via `iommu_fwspec` for cross-IOMMU integration.

MediaTek SMI runtime (per-IP):
1. Display driver `mtk_smi_larb_get(dev)`: pm-runtime-resume larb + bump consumer counter; first ref also resumes `smi_common`.
2. Per-port `MMU_EN(port)` bit programmed if IOMMU translation required for this DMA path.
3. `_put` reverses (`pm_runtime_put`, suspend larb if last ref).

Suspend/resume (TI EMIF via secure SRAM):
1. Bootloader stages `ti_emif_sram_pm.S` (asm code) into on-chip SRAM.
2. On suspend, kernel calls into secure firmware which uses the SRAM-resident asm to put DRAM in self-refresh + power off PHY.
3. On resume, secure firmware re-enters SRAM asm, restores PHY + leaves self-refresh.
4. Kernel post-resume re-runs `ti_emif_run_hw_leveling()` to re-train.

Interconnect provider (Tegra):
1. `tegra_mc_icc_xlate(of_args, data)` maps per-DT-binding stream-id → `icc_node`.
2. Client `of_icc_get(dev, NULL)` returns `icc_path`; `icc_set_bw(path, avg, peak)` updates per-client requested bandwidth.
3. Tegra MC aggregates per-client requests + programs EMC freq via paired EMC driver.

## Hardening

- Per-frequency `EmifRegs` blobs validated at probe (`emif_calculate_regs`); reject any frequency outside `min_tck`-derived bounds.
- Thermal IRQ handler skips reg rewrite if `temperature_level` unchanged; defense against IRQ flood.
- Tegra MC carveout-info reads validate range against `iomem_resource` to refuse a carveout overlapping kernel image.
- MediaTek SMI larb refcount is per-instance saturating; under/overflow trapped.
- ECC error reporting paths rate-limited via `__ratelimit` so a stuck-bit DRAM cannot DoS dmesg.
- DT parse paths in `of_memory.c` validate JEDEC-DDR-info length + per-rank count against DRAM-spec maxima before kmemdup.
- Suspend-path SRAM asm (`ti_emif_sram_pm.S`) is `__ro_after_init`-protected by hand-rolled section attributes.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `lpddr2_timings`, `emif_data`, `tegra_mc`, `tegra_mc_client`, `smi_larb`, per-frequency `EmifRegs`; debugfs reads bounded to fixed-size structs.
- **PAX_KERNEXEC** — memory-controller core text W^X; `tegra_mc_ops`, `emif_platform_data` ops, devfreq governor callbacks, and interconnect provider ops placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel stack on EMIF/IFC/MC IRQ entry, devfreq transition, SMI larb pm-runtime suspend/resume, and DT timing-parse entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on `mc`, `smi_larb`, `emif_data`, ECC error count; overflow trap defeats devfreq-race UAFs on shared timing blobs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-frequency timing blobs in `regs_cache`, ECC report records, and `lpddr2_*` parse buffers so stale DRAM characterization cannot leak across re-probe.
- **PAX_UDEREF** — SMAP/PAN enforced on every devfreq sysfs / debugfs write; userspace pointer deref bounded.
- **PAX_RAP / kCFI** — `tegra_mc_ops`, `emif_platform_data` ops, ECC report callbacks, devfreq governor hooks marked `__ro_after_init` with kCFI-typed indirect dispatch; suspend-resume asm-callable trampolines (`ti_emif_sram_pm.S`) verified at boot.
- **GRKERNSEC_HIDESYM** — gate kallsyms disclosure of MMIO bases, DRAM carveout addresses, and per-IP client pointers behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict ECC CE/UE banners, thermal-level changes, and devfreq transition spam to CAP_SYSLOG.
- **EMC sysfs CAP_SYS_ADMIN** — `/sys/class/devfreq/.../min_freq`, `max_freq`, `userspace/set_freq`, and any debugfs frequency-override on memory-controller devfreq instances require CAP_SYS_ADMIN.
- **Bandwidth-request PAX_USERCOPY** — `icc_set_bw` user-callable paths bounded; reject negative or > device-max bandwidths.
- **ECC error trace** — per-DIMM CE/UE records sanitized through EDAC's privileged interface (cross-ref `drivers/edac/`); raw physical address suppressed in non-CAP_SYSLOG dmesg.
- **Suspend/resume ops RAP** — `ti_emif_sram_pm` asm entry points have explicit hand-shaken signatures validated by the kernel before invocation; refuse mismatched sram-asm version.
- **Carveout-range validation** — `tegra_mc_get_carveout_info` returned ranges checked against kernel image, `iomem_resource.busy` children, and `memblock` reserved regions before exposing to client DMA path.
- **SMI larb power-state strict** — `mtk_smi_larb_get/put` strictly paired; refuse `_put` without prior `_get` (refcount underflow trap).

Rationale: SoC memory controllers configure the timing parameters that decide *whether DRAM actually contains the bits the CPU wrote*. A miswritten timing on a DVFS transition, a stale ECC report leaking physical addresses, or a misprogrammed carveout exposing kernel RAM to a non-IOMMU-protected display engine, all silently demote system integrity. Saturating refcounts on shared timing blobs, CAP_SYS_ADMIN on devfreq sysfs, EDAC-mediated ECC reporting, suspend-asm RAP signatures, and carveout-range cross-checks against `iomem_resource` and `memblock` keep the memory subsystem honest even across DVFS, suspend, and thermal transitions.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-SoC memory-controller drivers (`emif.c`, `fsl_ifc.c`, `mtk-smi.c`, `tegra/*`, `atmel-ebi.c`, `pl172.c`, `pl353-smc.c`, `omap-gpmc.c`, etc.) — Tier-4 each.
- EDAC integration for ECC report — separate Tier-3 (`drivers/edac/`).
- `drivers/interconnect/` framework itself — separate Tier-3.
- `devfreq` framework itself — separate Tier-3.
- Implementation code.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `timing_bounds_valid` | OOB | every `EmifRegs` blob's per-field value within JEDEC-spec range for its `lpddr2_timings.frequency`. |
| `carveout_no_overlap` | EXCLUSIVITY | Tegra carveout ranges disjoint from `iomem_resource.busy` children and kernel image. |
| `smi_larb_refcount_paired` | UAF | `_get`/`_put` pair correctly; refuse `_put` from underflow. |
| `ecc_trace_bounded` | RATE | ECC error records rate-limited; cannot fill dmesg ring. |

### Layer 2: TLA+

`models/memory/devfreq_transition.tla` (parent-declared): proves DVFS transition is atomic from clients' point of view — between `setup_registers` start and end, no client observes inconsistent timing; concurrent thermal IRQ during DVFS is serialized.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `emif_calculate_regs(emif, freq)` post: returned blob satisfies all timing inequalities from JEDEC tables | `emif::Emif::calculate_regs` |
| `mtk_smi_larb_get` post: larb power state == ON AND consumer count incremented exactly once | `mtk::SmiLarb::get` |
| `tegra_mc_write_emem_configuration(mc, rate)` post: programmed EMEM_CFG matches `rate`-derived params | `tegra::Mc::write_emem_configuration` |
| `ti_emif_restore_context` post: post-resume timing registers byte-equal to pre-suspend snapshot | `ti_emif::pm::restore_context` |

### Layer 4: Verus/Creusot functional

CPUfreq raises target → EMIF picks matching timing blob → `setup_registers` writes in JEDEC-mandated sequence → DRAM continues serving reads/writes without dropped transactions. Encoded as a Verus refinement of an abstract DVFS-monotonic-correctness specification.
