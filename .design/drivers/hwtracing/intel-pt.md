# Tier-3: drivers/hwtracing/intel_th/* + arch/x86/events/intel/pt.c — Intel Processor Trace + Trace Hub

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - arch/x86/events/intel/pt.c
  - arch/x86/events/intel/pt.h
  - drivers/hwtracing/intel_th/core.c
  - drivers/hwtracing/intel_th/gth.c
  - drivers/hwtracing/intel_th/msu.c
  - drivers/hwtracing/intel_th/msu-sink.c
  - drivers/hwtracing/intel_th/sth.c
  - drivers/hwtracing/intel_th/pti.c
  - drivers/hwtracing/intel_th/pci.c
  - drivers/hwtracing/intel_th/acpi.c
  - include/linux/intel_th.h
-->

## Summary

Intel Processor Trace (Intel PT) is the per-core CPU-instruction tracing engine introduced with Broadwell-E / Skylake; it is the closest x86 analogue of ARM ETMv4. The CPU streams compressed control-flow packets (TIP, TNT, FUP, MODE, PIP, PSB, MTC, CYC, PTW) to a destination programmed in IA32_RTIT_* MSRs, which is either a single physical-address output region or a Table-of-Physical-Addresses (ToPA) sg-list backing a perf-AUX ring buffer.

Intel Trace Hub (TH) is a chipset-side counterpart: a PCIe / ACPI device that aggregates trace sources (STH = System Trace, PTI = Parallel Trace Interface) into a Master Switch Unit (MSU) which can DMA into system RAM or push out a debug port; it is independent of Intel PT proper but shares the per-event MSU "ring buffer with ToPA" abstraction.

This Tier-3 covers `arch/x86/events/intel/pt.c` (~1893 lines: per-cpu PT PMU registered as `intel_pt`, ToPA table mgmt, perf AUX integration, `intel_pt_validate_cap`), `drivers/hwtracing/intel_th/core.c` (~1110 lines: TH bus + per-subdevice register/unregister), and `intel_th/msu.c` (~2204 lines: MSU sink with windowed multi-block ring + sg-list mmap).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pt` (per-cpu) | per-CPU PT state (handle, single/multiple-entry mode) | `drivers::intel_pt::PerCpu` |
| `struct pt_buffer` | per-event ring (ToPA tables + per-entry sg pages) | `drivers::intel_pt::Buffer` |
| `struct topa` / `struct topa_entry` | ToPA table + per-entry descriptor (page + END/INT/STOP bits) | `drivers::intel_pt::Topa` / `_TopaEntry` |
| `pt_pmu_init()` / `pt_pmu_event_init(event)` | per-cpu PMU init + per-event init | `Pmu::init` / `Pmu::event_init` |
| `pt_event_add(event, mode)` / `pt_event_del(...)` / `pt_event_start(event, flags)` / `pt_event_stop(...)` | PMU ops | `Pmu::*` |
| `pt_buffer_setup_aux(event, pages, nr_pages, snapshot)` / `pt_buffer_free_aux(data)` | per-event ToPA ring construction | `Pmu::setup_aux` / `_free_aux` |
| `topa_alloc(cpu, gfp)` / `topa_insert_pages(buf, cpu, gfp)` / `topa_insert_table(buf, cpu)` | per-page / per-table ToPA growth | `Buffer::insert_pages` / `_insert_table` |
| `pt_handle_status(pt)` | per-PMI status-bit decode (overflow, stop, etc.) | `PerCpu::handle_status` |
| `pt_event_stop` → IA32_RTIT_CTL.TraceEn=0 + IA32_RTIT_STATUS read-back | hard stop | `PerCpu::hard_stop` |
| `struct intel_th` | Trace Hub root device | `drivers::intel_th::TraceHub` |
| `struct intel_th_device` (subtype SOURCE/SWITCH/OUTPUT) | TH bus child (GTH/STH/MSU/PTI) | `drivers::intel_th::ThDevice` |
| `intel_th_driver_register(thdrv)` / `_unregister(thdrv)` | TH driver registration | `ThDriver::register` |
| `intel_th_alloc(dev, hw_ops, resource, nr_res)` / `intel_th_free(th)` | TH probe alloc | `TraceHub::alloc` |
| `intel_th_output_enable(th, otype)` / `intel_th_trace_enable/_switch/_disable(thdev)` | trace start/stop | `TraceHub::trace_enable` |
| `msu_buffer_register(mbuf)` / `_unregister(mbuf)` (msu.c) | per-MSU pluggable sink (single/multi-block) | `Msu::buffer_register` |

## Compatibility contract

REQ-1: CPUID feature detection: PT availability checked via CPUID leaf 14h; ToPA support (`topa_output`), multi-entry ToPA (`topa_multiple_entries`), psb_cyc, mtc, branch_en, ptwrite, power-event capabilities per `PT_CAP(...)` macro list.

REQ-2: Per-cpu PMU `intel_pt` registered with `perf_pmu_register`; `event->attr.config` mapped onto `RTIT_CTL_*` bits validated against PT_CONFIG_MASK + per-CPU feature mask via `pt_event_valid`.

REQ-3: ToPA table layout: each table is 4 KB containing 256 entries, last entry is END pointing to next table or wrap. Per-entry size is one of `TOPA_SIZE_4K..TOPA_SIZE_128M`; per-entry INT bit triggers PMI when filled; STOP bit halts.

REQ-4: Single-region vs ToPA modes: if `topa_output` capable, ToPA is preferred (multi-page sg ring); else legacy single-range (contiguous physical block).

REQ-5: Perf-AUX integration: `pt_buffer_setup_aux` allocates a `pt_buffer` covering `event->aux_pages` pages, builds ToPA tables placing one entry per AUX page; PMI on full triggers `perf_aux_output_*` to advance head.

REQ-6: Per-CPU MSR programming on `pt_event_start`: `IA32_RTIT_OUTPUT_BASE` = first ToPA phys, `IA32_RTIT_OUTPUT_MASK_PTRS` = packet-offset, `IA32_RTIT_CR3_MATCH` = filter CR3 (per-process trace), `IA32_RTIT_ADDR{0..3}_{A,B}` = IP-filter ranges, `IA32_RTIT_CTL` set last with TraceEn=1.

REQ-7: PMI handler decodes IA32_RTIT_STATUS: `Stopped`, `Error`, `ContextEn`, `TriggerEn`; updates AUX-output head + clears bits; re-enables trace if still active.

REQ-8: TH bus (`intel_th_bustype`) carries subdevices typed `INTEL_TH_SWITCH` (GTH = Global Trace Hub master mux), `INTEL_TH_SOURCE` (STH = System Trace, PTI), `INTEL_TH_OUTPUT` (MSU sink, MSU-sink "MSC" memory storage controller); per-device match by `intel_th_device_id`.

REQ-9: MSU "multi-window" ring: per-window sg-table of pages mmapped to userspace as a contiguous range via `remap_pfn_range` per page; per-window swap-out under host control.

REQ-10: PT supports per-CPU and per-thread modes. Per-thread mode programs CR3-match so packets only emit while the traced task is current; per-CPU mode traces unconditionally.

REQ-11: PMI rate-limiting: per-AUX-overflow + `STOP` bit ensures packet storms cannot livelock the CPU; perf framework rejects per-event config with overlapping address-filter ranges.

REQ-12: VMX/SGX/SMM filtering: PT respects per-mode VMX-host vs guest separation; SMM transitions discard packets to avoid leaking SMM IP.

## Acceptance Criteria

- [ ] AC-1: On Skylake+ HW, `perf record -e intel_pt//u -- ./victim` produces decodable trace via `perf script --insn-trace`.
- [ ] AC-2: ToPA multi-entry path: `perf record -e intel_pt// --aux-buffer-size=64M -a sleep 10` fills ring without packet loss outside expected `STOP` window.
- [ ] AC-3: IP-filter range: `perf record -e intel_pt//u --filter 'filter 0x400000/0x100000 @ ./victim'` produces packets confined to that IP window.
- [ ] AC-4: CR3 match for per-thread: per-task trace excludes context-switch noise from other tasks.
- [ ] AC-5: PMI overflow: stress with `STOP` bit cleared yields graceful AUX overrun event, never CPU lock-up.
- [ ] AC-6: TH driver enumerates GTH + STH + MSU on supported Intel platforms (e.g. Apollo Lake, Tiger Lake debug port); `/sys/bus/intel_th/devices/` lists them.
- [ ] AC-7: MSU mmap path: userspace reads MSU window via mmap; data agrees with debug-stream reference dump.
- [ ] AC-8: Hot-plug HW-pinned CPU removed mid-record → per-CPU PT cleanly disabled; AUX buffer truncated, not corrupted.

## Architecture

`PerCpu` PT state:

```
struct PerCpu {
  cpu: u32,
  handle_nmi: AtomicBool,        // currently in PMI
  buf: Option<KBox<Buffer>>,     // active per-event buffer
  filters: AddrFilters,          // per-event IP-filter cache
  handle: PerfOutputHandle,      // perf AUX-output handle
  vmx_on: AtomicBool,            // VMX state for guest-PT enable
  rtit_ctl: u64,                 // cached IA32_RTIT_CTL
  output_base: u64,              // cached IA32_RTIT_OUTPUT_BASE
  output_mask: u64,              // cached IA32_RTIT_OUTPUT_MASK_PTRS
}

struct Buffer {
  event: Arc<PerfEvent>,
  cpu: u32,
  snapshot: bool,
  pages: Vec<NonNull<Page>>,
  topa_list: Vec<KBox<Topa>>,
  last: Option<&mut TopaEntry>,
  cur: Option<&mut TopaEntry>,
  cur_idx: u32,
  output_off: u32,
  head: u64,                     // perf AUX head (bytes since start)
  nr_pages: u32,
  data_size: u64,
  lost: AtomicU64,
}

struct Topa {
  list: ListHead,
  offset: u32,                   // entries used
  size: u32,                     // total entries
  last: bool,
  phys: u64,                     // phys addr of this table page
}
```

PT per-event init `pt_event_init`:
1. Validate caller has CAP_SYS_ADMIN unless `attr.exclude_kernel` (per-event privilege model is the perf per-event paranoid level).
2. Decode `attr.config` against PT_CONFIG_MASK; reject reserved bits.
3. Validate PSB/MTC/CYC ranges against `pt_pmu.psb_periods`, `mtc_periods`, `cyc_thresholds` (from CPUID leaf 14h sub-leaf 1).
4. Validate address-filter ranges (max 4 per-CPU); reject overlapping ranges.
5. Resolve sink: PT writes into perf-AUX; ToPA tables built lazily at `setup_aux`.

ToPA setup `pt_buffer_setup_aux`:
1. Allocate `pt_buffer { pages, snapshot, head=0 }`.
2. Allocate first ToPA table (4 KB) via `topa_alloc(cpu, GFP_KERNEL | __GFP_ZERO)`.
3. For each AUX page: `topa_insert_pages` appends a `topa_entry { base = page_to_phys, size = TOPA_SIZE_4K, intr = (page_idx == nr_pages-1) }`; if current ToPA full, allocate next table and END-link previous.
4. Last table END-entry points back to first (circular) if `!snapshot`, else STOP-bit set.

Event start `pt_event_start`:
1. Disable per-CPU NMI watchdog wrt PT-PMI.
2. Sync per-cpu cached RTIT_CTL via `wrmsrl(IA32_RTIT_CTL, 0)` first to be safe.
3. Program RTIT_OUTPUT_BASE / OUTPUT_MASK_PTRS to first ToPA + cur offset.
4. Program CR3_MATCH per-event-task; program ADDR{0..3}_{A,B} per `event->addr_filter_ranges`.
5. Write IA32_RTIT_CTL with TraceEn=1 + config bits.
6. `perf_aux_output_begin(handle, event)` records starting head.

PMI handler `intel_pt_interrupt`:
1. Read IA32_RTIT_STATUS. If `Error`: clear, deliver `PERF_RECORD_AUX` with `PERF_AUX_FLAG_TRUNCATED`.
2. If `Stopped`: STOP-bit fired, compute bytes-since-head from current ToPA cur + offset, `perf_aux_output_end(handle, bytes)`.
3. Clear `Stopped`; if event still active, `perf_aux_output_begin` again on a fresh AUX region.
4. Acknowledge perf-NMI.

Intel TH bus:
1. Per-platform probe (`intel_th/pci.c` or `acpi.c`) discovers TH PCIe / ACPI device with N BARs.
2. `intel_th_alloc(dev, hw_ops, resource, nr_res)` builds `intel_th` root + scans subdevice descriptors.
3. Per subdevice (`intel_th_subdevice_alloc`) registered onto `intel_th_bustype` with type SWITCH/SOURCE/OUTPUT.
4. Per-driver match via `intel_th_driver` → `gth`, `sth`, `pti`, `msu`.

MSU buffer (msu.c): pluggable via `msu_buffer_register` for sinks like "MSC single-window", "MSC multi-window"; per-window mmap exposes captured bytes. PMI on window-full advances head; per-window switchable by userspace ioctl.

## Hardening

- **MSR-write strictness** — only the per-cpu PT code path may touch IA32_RTIT_* via `wrmsrl_safe`; wrappers reject writes when `pt_pmu.handle_nmi` is set in a context that does not own it.
- **ToPA entry validation** — per-entry phys must be `pfn_valid` + RAM-backed + not in kernel image / module text; reject anything else at `topa_insert_pages`.
- **Address-filter range validation** — `event->addr_filter_ranges` clamped to `[0, TASK_SIZE_MAX]`; refuse kernel addresses for unprivileged users.
- **PMI handler bounded** — per-invocation work cap; STOP-bit forces packet drain rather than spin.
- **VMX coexistence** — per-cpu PT disabled across VMX root-mode entry if guest also uses PT; refuse double-enable.
- **TH BAR mapping CAP_SYS_RAWIO** — `intel_th_alloc` resource map gated; defense against rogue probe of trace-hub MMIO.
- **MSU mmap range validated** — per-window mmap size matches sg-table; refuse OOB pages.
- **Per-CPU hot-unplug quiesce** — CPU offline notifier calls `pt_event_stop` for any active per-cpu event before CPU power-down.
- **Snapshot vs streaming separation** — snapshot mode marks STOP last entry so trace stops cleanly; streaming wraps via END-entry; mode-mismatch rejected at setup.
- **SMM filtering enforced** — IA32_RTIT_CTL.OS=1 path checks per-event paranoid level so user-only events never see SMM IP.
- **Per-PMU exclusion of unprivileged kernel trace** — if `attr.exclude_kernel == 0`, require `perf_event_paranoid <= 1` (kernel-default) AND CAP_PERFMON.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `pt_buffer`, `topa`, `topa_entry`, and TH `intel_th_device`; ToPA-entry copies in/out of userspace forbidden outside fixed-size sysfs helpers.
- **PAX_KERNEXEC** — PT PMU + Intel TH core in W^X kernel text; `pmu_ops`, `intel_th_driver`, `msu_buffer` vtables marked `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `pt_event_add`, `pt_event_start`, `pt_pmi`, and `msu_mmap` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `pt_buffer` per-event reference, ToPA-table list refcount, TH `intel_th` root refcount, and per-subdevice refcount.
- **PAX_MEMORY_SANITIZE** — zero-on-free for ToPA tables, AUX backing pages on event teardown, MSU window buffers, and per-event RTIT_* shadow registers so a prior tenant's trace cannot bleed.
- **PAX_UDEREF** — SMAP/PAN enforced on every TH sysfs / MSU mmap entry; reject user-pointer deref outside the perf AUX helpers.
- **PAX_RAP / kCFI** — `pmu_ops` for `intel_pt`, `intel_th_driver` vtable, MSU `msu_buffer` callbacks, and per-output `intel_th_output_ops` dispatched via kCFI-typed indirect calls; vtables `__ro_after_init`.
- **GRKERNSEC_HIDESYM** — gate disclosure of ToPA table phys addresses, MSU BAR base, and per-CPU PT register state in debugfs and tracepoints behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict PT overflow / TH probe banners to CAP_SYSLOG so attackers cannot side-channel trace activity via dmesg.
- **CAP_SYS_ADMIN strict** — `perf_event_open` for `intel_pt` with `exclude_kernel=0` requires CAP_PERFMON (kernel) + CAP_SYS_ADMIN (legacy paranoid mode); MSU ioctl window-switch requires CAP_SYS_ADMIN.
- **ToPA entry PAX_USERCOPY-bounded** — per-entry phys/size copies via fixed-typed accessors; OOB indexing trapped.
- **MSR write strict** — IA32_RTIT_* writes only via the per-cpu PT module; `msr-safe` denies userspace writes via /dev/cpu/N/msr.
- **Packet output PMI sanitize** — PMI handler validates ToPA cur index < `topa.size` before computing head; refuse to advance head if status indicates Error.
- **Per-event IP filter validated** — refuse kernel-range filters from non-CAP_SYS_ADMIN; refuse overlapping ranges that confuse decoder.

Rationale: Intel PT is a per-CPU instruction firehose. An unprivileged enable with `exclude_kernel=0` defeats KASLR and leaks every executed kernel branch. A maliciously crafted ToPA entry can DMA into arbitrary RAM. RAP/kCFI on PMU + TH vtables, capability gating on `perf_event_open` and MSU mmap, ToPA-entry validation, and MSR-write isolation turn Intel PT from a debug toy into a structurally fenced tracing facility.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- AMD IBS / AMD-LBR analogues (covered separately)
- ARM ETMv4 / CoreSight (covered in `coresight.md`)
- KVM-guest Intel PT virtualisation (covered in `arch/x86/kvm/` Tier-3 docs)
- TH platform glue per-SoC (covered in `intel-th-pci.md` / `intel-th-acpi.md` future)
- 32-bit-only paths
- Implementation code
