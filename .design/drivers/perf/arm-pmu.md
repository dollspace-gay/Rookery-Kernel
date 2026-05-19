# Tier-3: drivers/perf/{arm_pmu,arm_pmuv3}.c — ARM PMU framework + ARMv8 PMUv3 perf-event implementation (PMCR_EL0, per-CPU IRQ/NMI, chained counters, user-mode access, branch tracking)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/perf/arm_pmu.c
  - drivers/perf/arm_pmuv3.c
  - drivers/perf/arm_pmu_acpi.c
  - drivers/perf/arm_pmu_platform.c
  - drivers/perf/arm_brbe.c
  - include/linux/perf/arm_pmu.h
-->

## Summary

`drivers/perf/arm_pmu.c` is the generic per-CPU PMU framework used by every ARM core-PMU driver (`arm_v6_pmu`, `arm_v7_pmu`, `arm_xscale_pmu`, `arm_pmuv3` for ARMv8/ARMv9, `apple_m1_cpu_pmu` and friends). It provides the perf-events glue: `struct arm_pmu` allocation, per-CPU IRQ request and routing (legacy IRQ, percpu IRQ, percpu NMI), cpu-hotplug callbacks that re-establish the per-CPU PMU context on online/offline, `armpmu_dispatch_irq` shared interrupt handler, and the perf-core vtable (`event_init`, `add`, `del`, `start`, `stop`, `read`, `enable`, `disable`). `drivers/perf/arm_pmuv3.c` is the ARMv8 PMUv3 implementation — it owns PMCR_EL0 (per-CPU PMU control register), PMCNTENSET/CLR_EL0 (per-counter enable mask), PMOVSCLR_EL0 (overflow status), PMUSERENR_EL0 (user-mode access gate), PMSELR_EL0/PMXEVCNTR_EL0 (event counter selection), and per-counter PMEVCNTRn_EL0/PMEVTYPERn_EL0 indexed by counter number. PMUv3 also implements counter chaining (two adjacent 32-bit counters → one 64-bit virtual counter for `event->attr.config1 & ARMV8_PMU_LONG_EVENT_BIT`), per-event filtering (USER/KERNEL/HYPERVISOR/SECURE state masks in PMEVTYPERn), threshold counters (ARMv8.8+ count-on-condition), and (via `arm_brbe.c`) Branch Record Buffer Extension when the part implements it. The framework registers as a perf `struct pmu` instance shared across the cpumask the PMU covers (typically `cpu_possible_mask`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct arm_pmu` | per-CPU-PMU descriptor (pmu, name, num_events, supported_cpus, …) | `drivers::arm_pmu::ArmPmu` |
| `struct pmu_hw_events` | per-CPU counter assignment state (events[], used_mask, …) | `drivers::arm_pmu::HwEvents` |
| `struct pmu_irq_ops` | per-IRQ-kind ops vtable (free, enable, disable) | `drivers::arm_pmu::IrqOps` |
| `armpmu_alloc()` / `armpmu_free(pmu)` | per-CPU `arm_pmu` allocate/free | `ArmPmu::alloc` / `_free` |
| `armpmu_register(pmu)` | register with perf core | `ArmPmu::register` |
| `armpmu_event_init(event)` | validate event vs PMU; assign counter index | `ArmPmu::event_init` |
| `armpmu_add(event, flags)` / `_del(event, flags)` | assign / release counter | `ArmPmu::add` / `_del` |
| `armpmu_start(event, flags)` / `_stop(event, flags)` / `_read(event)` | enable/disable/read counter | `ArmPmu::start` / `_stop` / `_read` |
| `armpmu_event_set_period(event)` | program counter sample period | `ArmPmu::set_period` |
| `armpmu_dispatch_irq(irq, dev)` | shared per-CPU IRQ handler | `ArmPmu::dispatch_irq` |
| `armpmu_request_irq(pcpu_armpmu, irq, cpu)` / `_free_irq(...)` | per-CPU IRQ / percpu-IRQ / NMI request | `ArmPmu::request_irq` / `_free_irq` |
| `cpu_pm_pmu_notify(b, cmd, v)` | CPU PM-suspend save/restore | `ArmPmu::pm_notify` |
| `arm_perf_starting_cpu(cpu, node)` / `_teardown_cpu(...)` | CPU-hotplug callbacks | `ArmPmu::hp_starting` / `_hp_teardown` |
| `arm_pmu_acpi_probe(init_fn)` / `arm_pmu_device_probe(pdev, of_table, init_fn)` | ACPI / DT probe entry | `Subsystem::acpi_probe` / `_dt_probe` |
| `armv8pmu_pmcr_read()` / `armv8pmu_pmcr_write(val)` | PMCR_EL0 r/w | `Pmuv3::pmcr_read` / `_pmcr_write` |
| `armv8pmu_read_counter(event)` / `_write_counter(event, value)` | per-event counter access | `Pmuv3::read_counter` / `_write_counter` |
| `armv8pmu_enable_counter(mask)` / `_disable_counter(mask)` | PMCNTENSET/CLR | `Pmuv3::enable_counter` / `_disable_counter` |
| `armv8pmu_enable_intens(mask)` / `_disable_intens(mask)` | PMINTENSET/CLR (overflow IRQ enable) | `Pmuv3::enable_intens` / `_disable_intens` |
| `armv8pmu_write_evtype(idx, val)` | PMEVTYPERn (event code + filter bits) | `Pmuv3::write_evtype` |
| `armv8pmu_has_long_event(pmu)` / `_event_is_chained(event)` | 64-bit counter chaining | `Pmuv3::has_long_event` / `_event_is_chained` |
| `armv8pmu_event_threshold_control(attr)` / `_event_get_threshold(attr)` | ARMv8.8 threshold counters | `Pmuv3::event_threshold_*` |
| `sysctl_perf_user_access` | sysctl gating PMUSERENR_EL0 EN/CR/SW/ER bits | `Pmuv3::user_access_sysctl` |

## Compatibility contract

REQ-1: `arm_pmu` is per-CPU-PMU (not per-CPU) — one `arm_pmu` instance may cover multiple CPUs (homogeneous cluster) via `supported_cpus` cpumask; heterogeneous (big.LITTLE) → one instance per uarch cluster.

REQ-2: `armpmu_alloc()` allocates `pmu` plus a per-CPU `pmu_hw_events` array; `armpmu_register` calls `perf_pmu_register(&pmu->pmu, name, PERF_TYPE_RAW)` with `cpumask` attribute populated from `supported_cpus`.

REQ-3: IRQ kinds: legacy (one IRQ per CPU, requested via `request_irq` + `IRQF_NOBALANCING`), per-cpu (single irq number, `request_percpu_irq`), per-cpu NMI (`request_percpu_nmi`, gated by `arm_pmu_irq_is_nmi`). Per-CPU ops dispatched through `pmu_irq_ops`.

REQ-4: Per-CPU IRQ delivery model: when an overflow interrupt fires on CPU N, `armpmu_dispatch_irq` walks `cpuc->used_mask`, reads `PMOVSCLR_EL0`, dispatches each overflowed counter's bound event to `perf_event_overflow`, and clears the overflow bit.

REQ-5: CPU PM notifier (`cpu_pm_pmu_notify`): on `CPU_PM_ENTER` save PMCR + per-counter enable/event masks; on `CPU_PM_EXIT` restore. This covers idle entries that lose PMU state.

REQ-6: CPU hotplug (`arm_perf_starting_cpu` / `_teardown_cpu`): on starting CPU, re-arm PMUSERENR + clear PMCR; on teardown, stop counters and decouple from the per-CPU `arm_pmu` pointer.

REQ-7: PMUv3 counter inventory: PMCR_EL0.N reports the number of generic event counters (0..30); counter index 31 is the dedicated CPU-cycle counter (PMCCNTR_EL0). 32-bit wide unless ARMv8.5+ where long-event support extends to 64 bits.

REQ-8: Counter chaining (pre-ARMv8.5): a 64-bit virtual counter is two adjacent 32-bit counters (idx, idx+1); `armv8pmu_event_is_chained(event)` detects the chain; chain root holds low half, chain-high counter chains-via `ARMV8_PMUV3_PERFCTR_CHAIN` event.

REQ-9: Event filter bits in PMEVTYPERn: U (count in EL0), NSU (count in non-secure EL0), K (count in EL1), NSK, M (count in EL3), MT (multi-threaded), SH, T (count in EL2 — hypervisor). Defaults exclude EL2/EL3 unless attr requests.

REQ-10: User-mode counter access (PMUSERENR_EL0): the kernel exposes EN/CR/SW/ER bits per the `sysctl_perf_user_access` sysctl (`/proc/sys/kernel/perf_user_access`); when enabled, user-space can read PMCCNTR_EL0 / PMEVCNTRn_EL0 directly via the rdpmc-equivalent path that perf userspace exposes through mmap'd `perf_event_mmap_page`.

REQ-11: Threshold-counter support (ARMv8.8+): `event->attr.config1` bits encode threshold + comparison direction (`armv8pmu_event_get_threshold`, `_event_threshold_control`); `threshold_max` is exposed via `caps` sysfs.

REQ-12: Branch Record Buffer Extension (`drivers/perf/arm_brbe.c`): when supported and CONFIG_ARM_BRBE_PMU=y, per-event branch records are exposed under `BRANCH_STACK` sample type.

## Acceptance Criteria

- [ ] AC-1: On an ARMv8 board, `ls /sys/bus/event_source/devices/armv8_pmuv3/events/` lists named PMUv3 events (cpu_cycles, l1d_cache, br_mis_pred, …).
- [ ] AC-2: `perf stat -e cpu-cycles,instructions,branches,branch-misses sleep 1` returns non-zero values for every counter on a single CPU.
- [ ] AC-3: `perf stat -a -e cpu-cycles sleep 1` aggregates across all CPUs in `supported_cpus`.
- [ ] AC-4: On big.LITTLE (e.g. Cortex-A55+A76), `/sys/bus/event_source/devices/` shows separate PMU instances per cluster with disjoint cpumasks.
- [ ] AC-5: PMU IRQ test: with `perf record -e cpu-cycles -c 100000 ...`, every CPU's IRQ counter in `/proc/interrupts` increments.
- [ ] AC-6: User-access test: `echo 1 > /proc/sys/kernel/perf_user_access`; a user-space test program that maps `perf_event_mmap_page` reads PMCCNTR via `rdpmc` semantics; with `echo 0`, the same op faults.
- [ ] AC-7: CPU hotplug: `echo 0 > /sys/devices/system/cpu/cpu1/online; echo 1 > .../online` does not leak counters and `perf stat -a` on the second iteration counts the re-onlined CPU.
- [ ] AC-8: Suspend/resume cycle: counters resume programmed values after S2idle.
- [ ] AC-9: kselftest `tools/testing/selftests/arm64/pmu/` (where present) passes.

## Architecture

`ArmPmu` in `drivers::arm_pmu::ArmPmu`:

```
struct ArmPmu {
  pmu: PerfPmu,                          // registered with perf core
  name: KStr,
  num_events: u32,                       // generic event counters available
  supported_cpus: Cpumask,
  hw_events: PerCpu<HwEvents>,           // per-CPU counter assignment
  pmceid: [u64; 4],                      // event implementation bitmaps
  acpi_cpuid: u64,
  cpu_pm_nb: NotifierBlock,
  hp_node: HlistNode,                    // cpuhp state attachment

  // function pointers (PMUv3 backend installs them):
  handle_irq: fn(&Self) -> IrqReturn,
  enable: fn(&PerfEvent),
  disable: fn(&PerfEvent),
  get_event_idx: fn(&HwEvents, &PerfEvent) -> i32,
  clear_event_idx: fn(&HwEvents, &PerfEvent),
  set_event_filter: fn(&mut HwPerfEvent, &PerfEventAttr) -> Result<()>,
  read_counter: fn(&PerfEvent) -> u64,
  write_counter: fn(&PerfEvent, u64),
  start: fn(&Self), stop: fn(&Self),
  reset: fn(*const c_void),
  map_event: fn(&PerfEvent) -> Result<i32>,
}

struct HwEvents {
  events: [Option<Arc<PerfEvent>>; ARMPMU_MAX_HWEVENTS],
  used_mask: BitMask<ARMPMU_MAX_HWEVENTS>,
  irq: Option<u32>,
}
```

Event init `ArmPmu::event_init(event)`:
1. Validate `event->attr.type == pmu.type`.
2. `map_event(event)` returns hardware event code (PMUv3 has a static event map; raw events via `event->attr.config` accepted in range).
3. Build `hw->config_base` from filter bits per `event->attr.exclude_*`.
4. `armpmu_event_set_period(event)` computes initial counter sample period from `event->attr.sample_period`.
5. Return 0 (counter assignment deferred to `add`).

Counter assign `ArmPmu::add(event, flags)`:
1. `idx = get_event_idx(cpuc, event)`:
   - PMUv3 first tries cycle-counter (idx 31) if event is `cpu-cycles`.
   - Otherwise scan `used_mask` for free slot < `num_events`.
   - For chained events, find a pair of adjacent free slots.
2. Set `used_mask[idx]` (and `idx+1` if chained); store `event` in `events[idx]`.
3. If `flags & PERF_EF_START`, call `start(event, PERF_EF_RELOAD)`.

Counter start `Pmuv3::enable_event` (via `arm_pmu::enable`):
1. `write_evtype(idx, hw->config_base)` (PMEVTYPERn = event code | filter bits).
2. `armpmu_event_set_period(event)` writes initial counter value via `write_counter`.
3. `enable_counter(BIT_ULL(idx))` (PMCNTENSET).
4. `enable_intens(BIT_ULL(idx))` (PMINTENSET) for sampling events.

Counter read `Pmuv3::read_counter(event)`:
1. `idx = event->hw.idx`.
2. If `armv8pmu_event_has_user_read(event)` AND `sysctl_perf_user_access` enabled: rely on user-space `rdpmc` semantics; kernel-side read still happens for `perf_event->count` accounting.
3. `value = read PMEVCNTRn` (or `PMCCNTR` for idx 31).
4. If chained: `value = (read PMEVCNTRn) | (read PMEVCNTRn+1 << 32)`.
5. Long-event bias: `armv8pmu_bias_long_counter`/`_unbias_long_counter` keep arithmetic consistent across the 32→48→64-bit counter widths the spec allows.

IRQ dispatch `ArmPmu::dispatch_irq`:
1. `pmovsr = read PMOVSCLR_EL0`.
2. If `pmovsr == 0`: return `IRQ_NONE` (shared-IRQ false-positive).
3. `pmu_stop()` (clear PMCR.E) — freeze counters during processing.
4. For each `idx` in `pmovsr`:
   - `event = cpuc->events[idx]`.
   - `armpmu_event_update(event)` — accumulate counter delta into `event->count`.
   - `perf_sample_data_init(&data, 0, hwc->last_period)`.
   - `if (perf_event_overflow(event, &data, regs)) armpmu_stop(event, 0)`.
5. Clear `pmovsr` via PMOVSCLR_EL0 write.
6. `pmu_start()` — resume counters.
7. Return `IRQ_HANDLED`.

IRQ request `ArmPmu::request_irq` per-CPU:
1. Determine kind: legacy / percpu / percpu-NMI based on `arm_pmu_irq_is_nmi` and platform.
2. For NMI: `request_percpu_nmi(irq, armpmu_dispatch_irq, "arm-pmu", percpu_armpmu)` — enables NMI overflow handling so high-frequency sampling does not interleave with IRQ-disabled regions.
3. For legacy: `request_irq(irq, dispatch_irq, IRQF_NOBALANCING | IRQF_NO_THREAD, "arm-pmu", percpu_armpmu)`.

ACPI probe `Subsystem::acpi_probe(init_fn)` (`arm_pmu_acpi.c`): walks MADT GICC entries to discover per-CPU GSI for PMU IRQ; calls `init_fn(pmu)` (typically `armv8_pmuv3_init`) to populate the PMUv3 function pointers and event map.

DT probe `Subsystem::dt_probe(pdev, of_table, init_fn)` (`arm_pmu_platform.c`): parses `interrupts` + `interrupt-affinity` properties; for percpu IRQ topology uses `pmu_parse_percpu_irq`.

## Hardening

- PMCR/PMCNTENSET/PMUSERENR writes are kernel-only via privileged register access — userspace cannot touch them without the kernel installing PMUSERENR.EN.
- `sysctl_perf_user_access` is the *only* path that allows userspace to read PMU counters directly; default is 0 (off) and the sysctl write requires `CAP_SYS_ADMIN`.
- Per-event filter bits are validated against `attr.exclude_*`; the kernel refuses to set EL2/EL3 count bits unless the caller has the appropriate capability.
- Counter index allocation is bounded by `num_events` plus the dedicated cycle counter (idx 31); `used_mask` is a hard bitmap so no double-assignment is possible.
- IRQ handler freezes PMCR.E before walking overflows so a concurrent counter update cannot race with `perf_event_overflow`.
- CPU PM save/restore is bounded — only the live counters' PMEVTYPERn/PMEVCNTRn are saved, not the entire register file.
- DT/ACPI probe validates per-CPU IRQ ownership: an IRQ may be claimed by at most one PMU instance; conflicts are refused.
- Branch Record Buffer (BRBE) entries are filtered against EL0/EL1 sampling permission before being exposed via `BRANCH_STACK`.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `arm_pmu`, `pmu_hw_events`, `pmu_irq_ops`, per-event `hw_perf_event` extensions, and BRBE record buffers; `perf_event_mmap_page` updates cross only the perf-core USERCOPY path.
- **PAX_KERNEXEC** — `arm_pmu` function-pointer table, `pmu_irq_ops` (`pmuirq_ops`, `pmunmi_ops`, `percpu_pmuirq_ops`, `percpu_pmunmi_ops`), and the registered `struct pmu` callbacks live in W^X regions; `__ro_after_init` after `armpmu_register`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `armpmu_dispatch_irq`, `armpmu_event_init`, `armpmu_add`, and the CPU-PM notifier entry to break leaks of `hw_events` pointers.
- **PAX_REFCOUNT** — saturating `refcount_t` on `arm_pmu` and on perf-event references held by `used_mask` assignment; overflow trap defeats hotplug/teardown races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `arm_pmu`, `pmu_hw_events`, BRBE record buffers, and per-CPU saved PMU state so stale counter assignments and branch records cannot bleed across CPU online/offline cycles.
- **PAX_UDEREF** — SMAP/PAN enforced on `perf_user_access` sysctl writers and on every perf-event syscall path that reaches `armpmu_event_init`; reject user pointers outside canonical helpers.
- **PAX_RAP / kCFI** — `arm_pmu` callback table, `pmu_irq_ops`, `struct pmu` (registered with perf core), and CPU-PM notifier block marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `arm_pmu`, `hw_events`, BRBE-buffer-base, and per-CPU IRQ ops pointers behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict PMU-init, IRQ-acquire-fail, CPU-PM-restore-fail, and BRBE-overflow banners to CAP_SYSLOG so attackers cannot probe PMU topology via dmesg.
- **PMCR_EL0 write CAP_SYS_ADMIN** — the global PMU enable/reset path is reachable only from kernel context; `sysctl_perf_user_access` writes (which transitively change PMUSERENR_EL0) require CAP_SYS_ADMIN against the owning user namespace.
- **Event-filter (allowed counter mask)** — `set_event_filter` validates `attr.exclude_*` against the calling task's privilege; EL2/EL3 count bits require CAP_PERFMON + CAP_SYS_ADMIN; refuse silent demotion of filters.
- **Branch-tracking CAP_SYS_ADMIN** — `BRANCH_STACK` sample type, BRBE configuration, and `arm_brbe.c` register touch require CAP_SYS_ADMIN; otherwise refuse the event registration so branch-record-driven side channels are not exposed to sandboxes.
- **vCPU PMU passthrough** — when KVM passes the host PMU into a guest, the driver refuses overlap with host-userspace perf events (mutual exclusion enforced via `armpmu_filter`); a guest with PMU access cannot poison host counters.
- **Kernel-counter expose gate** — userspace direct counter reads (`rdpmc` semantics via `perf_event_mmap_page`) are gated by `sysctl_perf_user_access == 1` *and* the event's exclude_kernel attribute; default policy refuses kernel-mode counter reads from userspace even when user access is enabled.

Rationale: the ARM PMU is simultaneously perf's hot path on every ARM platform and a known microarchitectural side-channel surface (cycle-counter granularity is sufficient to extract cryptographic keys via prime-and-probe). RAP/kCFI on `arm_pmu` callbacks and `pmu_irq_ops`, CAP_SYS_ADMIN gating of PMCR writes and `perf_user_access`, kCFI-typed `struct pmu` install, refcount-overflow trapping on PMU lifetime, refuse-on-conflict event-filter validation, branch-record gating, and zero-on-free of saved CPU-PM state turn ARM PMU from "any container with perf access can time your AES round" into a privilege-checked, side-channel-aware performance counter substrate.

## Open Questions

- The default value of `sysctl_perf_user_access` should be 0 on hardened images; consider exposing a kernel cmdline `armpmu.user_access=off` that survives `sysctl -w`. Out of scope here.
- BRBE record exposure to non-root currently relies on perf event paranoia level; tightening to require CAP_SYS_ADMIN regardless of `kernel.perf_event_paranoid` is an open policy decision.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `counter_idx_unique` | UNIQUENESS | `used_mask` bits are flipped atomically; no two events share a counter idx on the same CPU. |
| `irq_handler_no_oob` | OOB | `pmovsr` iteration bounded by `num_events`; `events[idx]` access stays within `ARMPMU_MAX_HWEVENTS`. |
| `pmcr_save_restore_balanced` | INVARIANT | every `CPU_PM_ENTER` save has a matching `CPU_PM_EXIT` restore; PMCR_EL0 bits identical pre/post idle. |
| `chained_counter_pair` | INVARIANT | for any chained event, `used_mask[idx] && used_mask[idx+1]`; never partial. |

### Layer 2: TLA+

`models/arm_pmu/irq_overflow.tla`: model concurrent counter overflows on N CPUs with shared-IRQ delivery; prove `armpmu_dispatch_irq` either handles every overflow or returns `IRQ_NONE`, with no missed `perf_event_overflow` callback and no double-handling of the same overflow.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `ArmPmu::add` post: `cpuc->events[idx] == event && used_mask[idx]` (and `used_mask[idx+1]` if chained) | `ArmPmu::add` |
| `ArmPmu::del` post: `cpuc->events[idx] == NULL && !used_mask[idx]` | `ArmPmu::del` |
| `Pmuv3::read_counter` post: returned value monotonically increasing modulo counter width (or chain width) | `Pmuv3::read_counter` |
| `cpu_pm_pmu_notify(CPU_PM_EXIT)` post: PMCR_EL0 + per-counter enable mask == saved snapshot | `ArmPmu::pm_notify` |

### Layer 4: Verus/Creusot functional

Userspace `perf_event_open(attr=cpu-cycles, ...)` → `armpmu_event_init` validates and maps event → `armpmu_add` allocates counter idx → `armpmu_start` programs PMEVTYPER + PMEVCNTR + enables → on overflow, `armpmu_dispatch_irq` delivers `PERF_RECORD_SAMPLE` to ring buffer → `read()` returns accumulated count via `armpmu_read`. Encoded as a chain of Verus invariants tying syscall-attr to counter program through to per-sample delivery, with overflow IRQ as the linchpin contract.
