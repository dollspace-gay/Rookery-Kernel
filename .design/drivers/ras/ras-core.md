# Tier-3: drivers/ras/{ras,cec,debugfs}.c — RAS subsystem (trace events, CEC, AMD-ATL decoder)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/ras/00-overview.md
upstream-paths:
  - drivers/ras/ras.c
  - drivers/ras/cec.c
  - drivers/ras/debugfs.c
  - drivers/ras/debugfs.h
  - drivers/ras/amd/atl
  - include/linux/ras.h
  - include/ras/ras_event.h
-->

## Summary

`drivers/ras/` is the kernel's central nexus for Reliability / Availability / Serviceability hooks: it owns the canonical RAS tracepoints (`mc_event`, `arm_event`, `non_standard_event`, `extlog_mem_event`, `aer_event`, `memory_failure_event`, `mf_action_result`), the Correctable-Errors Collector (CEC — a frequency-pressure poison-on-threshold filter for DRAM CECs that would otherwise spam logs without action), the per-`debugfs/ras` introspection surface, and the AMD-ATL address-translation library indirection (AMD UMC machine-check-bank address → system physical address translation, registered by the modular `amd-atl` decoder so that EDAC + MCE consumers can attribute errors to specific DIMM addresses on Zen 3+ silicon).

Upstream producers feed this subsystem from many directions: x86 MCE handlers, EDAC core, GHESv2 / APEI firmware-first error logs, PCIe AER, ACPI ExtLog, ARM SEA/SEI/SDEI, memory_failure handler. Consumers are `rasdaemon`, `mcelog`, `kdump` triage, on-line DIMM-replacement policy daemons.

This Tier-3 covers `ras.c` (~130 lines: tracepoint creation site, AMD-ATL decoder registration, ARM-event marshalling, subsys_initcall), `cec.c` (~610 lines: array-of-u64 CEC bookkeeping + decay + poison-threshold trigger), and `debugfs.c`/`debugfs.h` (per-cec controls and inspection).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `trace_mc_event(...)` / `trace_extlog_mem_event(...)` / `trace_non_standard_event(...)` / `trace_arm_event(...)` / `trace_aer_event(...)` / `trace_memory_failure_event(...)` / `trace_mf_action_result(...)` | RAS tracepoints created in `ras.c` | `drivers::ras::Tracepoints` |
| `log_non_standard_event(sec_type, fru_id, fru_text, sev, err, len)` | wrapper used by GHES non-standard error path | `Ras::log_non_standard_event` |
| `log_arm_hw_error(err, sev)` | ARM CPER section parser → `trace_arm_event` | `Ras::log_arm_hw_error` |
| `amd_atl_register_decoder(fn)` / `amd_atl_unregister_decoder()` / `amd_convert_umc_mca_addr_to_sys_addr(err)` | indirection to optional `amd-atl` module | `AmdAtl::register_decoder` / `_unregister_decoder` / `_convert` |
| `parse_ras_param(str)` — `ras=cec_disable` cmdline | boot-time RAS knobs | `Ras::parse_param` |
| `ras_debugfs_init()` | `/sys/kernel/debug/ras/` root | `RasDebugfs::init` |
| `ras_add_daemon_trace()` | enable-on-boot tracepoint for rasdaemon's default consumption set | `Ras::add_daemon_trace` |
| `cec_add_elem(pfn)` | CEC insert path called from `mce_amd`/`mce_intel`/`memory_failure` correctable path | `Cec::add_elem` |
| `cec_init()` / `parse_cec_param(str)` / `cec_decay_count` / `cec_action_threshold` | CEC config + lifecycle | `Cec::init` / `_parse_param` |
| `struct ce_array` (u64-per-element bit-packed PFN/decay/count) | CEC core data structure | `drivers::ras::Cec::Array` |
| `__find_elem(ca, pfn, *idx)` / `do_spring_clean(ca)` / `del_elem(ca, idx)` / `find_array_entry(ca, pfn)` | CEC internals | `Cec::find_elem` / `_spring_clean` / `_del_elem` / `_find_array_entry` |
| debugfs files `count_threshold`, `decay_interval`, `action_threshold`, `pfn`, `array_dump`, `disable` | per-CEC tuning + introspection | `RasDebugfs::*` |
| `struct atl_err` (`/include/linux/ras.h`) | AMD UMC MCA error descriptor passed into ATL decoder | `drivers::ras::AtlErr` |

## Compatibility contract

REQ-1: `ras_init` runs at `subsys_initcall`: registers per-RAS tracepoints, creates `/sys/kernel/debug/ras/`, conditionally instantiates CEC.

REQ-2: tracepoints registered in `ras.c` are the canonical kernel surface for rasdaemon to consume; `mc_event` (EDAC mem-controller event), `non_standard_event` (CPER non-std), `extlog_mem_event` (ACPI ExtLog memory), `arm_event` (ARM CPER processor), `aer_event` (PCIe AER), `memory_failure_event` / `mf_action_result` (memory-failure decision + outcome).

REQ-3: AMD-ATL indirection: at most one decoder registered at any time; once `amd_atl_umc_na_to_spa` is set it is not expected to be unset except for testing; `amd_convert_umc_mca_addr_to_sys_addr` returns `-EINVAL` when no decoder is registered.

REQ-4: CEC bit layout `[63 ... PFN ... 12 | 11..10 DECAY | 9..0 COUNT]` exactly fitted to PAGE_SHIFT==12 — the layout is page-shift-derived and verified at boot; on 16K PAGE_SHIFT architectures, `DECAY_BITS=2` and `COUNT_BITS=(PAGE_SHIFT - DECAY_BITS)` adjust.

REQ-5: CEC capacity is exactly `PAGE_SIZE / sizeof(u64)` entries per page; spring-cleaning is triggered once `decay_count` reaches `CLEAN_ELEMS = MAX_ELEMS >> DECAY_BITS`.

REQ-6: per-page CEC element retains LRU-ish ordering by virtue of generation bits being topped up on access (set to `0b11` on every insertion / increment); spring cleaning decrements all generations; full-decay (gen == 0) elements are evictable.

REQ-7: per-element `count` saturates at `action_threshold` (default `COUNT_MASK = (1 << COUNT_BITS) - 1`); when reached, the page is poisoned via `memory_failure` (PFN → physical-page-take-offline); `cec_add_elem` returns non-zero so caller logs the event externally too.

REQ-8: CEC is opt-in via `CONFIG_RAS_CEC=y` and runtime-disable via `ras=cec_disable` cmdline; once disabled it cannot be re-enabled at runtime (boot-time decision).

REQ-9: ARM CPER section parsing (`log_arm_hw_error`) bounds-checks every embedded `cper_arm_err_info` / `cper_arm_ctx_info` against `err->section_length`; firmware-supplied lengths inconsistent with declared section_length log `FW_BUG` and clamp the venor-error portion to zero rather than overrun.

REQ-10: debugfs surface (`/sys/kernel/debug/ras/`): `count_threshold` (read/write u32, action_threshold), `decay_interval` (read/write), `pfn` (`echo PFN > pfn` injects a synthetic CEC entry for test), `array_dump` (read: PFN/gen/count tuples), `disable` (read-only).

REQ-11: `cec_add_elem` is called from MCE hot paths and is IRQ-safe to a degree — it uses a mutex but is invoked from a kthread / softirq context already; not callable from hardirq.

REQ-12: ARM-event tracepoint emits `cpu` derived via `GET_LOGICAL_INDEX(err->mpidr)` mapping the firmware-supplied MPIDR to the logical CPU id; `-1` if unknown.

## Acceptance Criteria

- [ ] AC-1: `cat /sys/kernel/debug/tracing/events/ras/mc_event/format` exists; `mcelog` / `rasdaemon` consumes events without parse error on a system with ECC DRAM under EDAC.
- [ ] AC-2: CEC injection: `echo 0x12345 > /sys/kernel/debug/ras/pfn` repeated until `action_threshold`; `memory_failure` invoked → `/proc/meminfo` `HardwareCorrupted` increments; the corresponding 4 KiB page taken offline.
- [ ] AC-3: AMD-ATL: on Zen 3+ system with `amd_atl` modprobed, `amd_convert_umc_mca_addr_to_sys_addr` translates a known UMC normalized-address sample to the documented SPA.
- [ ] AC-4: ARM CPER: synthetic GHES record exercising every `cper_arm_err_info` / `cper_arm_ctx_info` count yields a single `arm_event` trace with consistent PEI / CTX / VSEI splits; no `FW_BUG` on valid input.
- [ ] AC-5: `ras=cec_disable` cmdline path: `/sys/kernel/debug/ras/disable` reads `Y`; `cec_add_elem` is a no-op.
- [ ] AC-6: spring-clean: insert > CLEAN_ELEMS entries to force a sweep; `decays_done` increments; no leaked-element growth (`n <= MAX_ELEMS`).
- [ ] AC-7: KASAN clean across `array_dump` read concurrent with `cec_add_elem` insertion stream.
- [ ] AC-8: extlog: ACPI ExtLog memory error path emits `extlog_mem_event` consumed by rasdaemon; tracepoint EXPORT_SYMBOL gated `CONFIG_ACPI_EXTLOG`.
- [ ] AC-9: ARM-event mpidr→cpu mapping returns valid logical id on a multicore ARM target with online CPUs and `-1` on an offline CPU.

## Architecture

`Cec` lives in `drivers::ras::Cec`:

```
struct CeArray {
  array: NonNull<u64>,             // single PAGE_SIZE-sized container
  n: u32,                          // number of live elements
  decay_count: u32,                // insertions/increments since last spring-clean
  pfns_poisoned: u64,
  ces_entered: u64,
  decays_done: u64,
  disabled: bool,
}

const MAX_ELEMS: usize     = PAGE_SIZE / 8;
const DECAY_BITS: u32      = 2;
const DECAY_MASK: u64      = (1 << DECAY_BITS) - 1;
const COUNT_BITS: u32      = PAGE_SHIFT - DECAY_BITS;
const COUNT_MASK: u64      = (1 << COUNT_BITS) - 1;
const CLEAN_ELEMS: usize   = MAX_ELEMS >> DECAY_BITS;
```

CEC insert `cec_add_elem(pfn)`:
1. Bail if `ca.disabled || !ca.array`.
2. `find_elem(ca, pfn, &idx)` binary-search by PFN (array kept sorted by PFN).
3. If found:
   - Bump count; clamp at COUNT_MASK; refresh DECAY to `0b11`.
   - If `count == action_threshold`: call `memory_failure(pfn, 0)`; `del_elem(ca, idx)`; bump `pfns_poisoned`.
4. Else (new entry):
   - If `n == MAX_ELEMS`: `do_spring_clean(ca)`; if still no room, evict oldest gen-0 entry.
   - `memmove(ca->array+idx+1, ca->array+idx, (n-idx)*8)`; insert `(pfn<<PAGE_SHIFT) | (0b11 << COUNT_BITS) | 1`.
   - `n += 1`.
5. `decay_count += 1`; if `decay_count >= CLEAN_ELEMS`, `do_spring_clean(ca)`.
6. Return non-zero if event reached threshold (signals caller to log externally).

Spring-clean `do_spring_clean(ca)`:
1. Iterate array; for each element, decrement DECAY bits; if DECAY underflows past zero, mark for eviction.
2. Compact array via memmove; `n` reduces; `decays_done += 1`; `decay_count = 0`.

ARM CPER processing `log_arm_hw_error(err, sev)`:
1. `pei_len = sizeof(cper_arm_err_info) * err->err_info_num`; `pei_err = (u8 *)(err+1)`.
2. Walk `err->context_info_num` `cper_arm_ctx_info` structs; compute `ctx_len`; each entry's variable-length tail clamped to `section_length`.
3. `vsei_len = section_length - (sizeof(cper_sec_proc_arm) + pei_len + ctx_len)`; if negative, dump `FW_BUG` + zero.
4. `cpu = GET_LOGICAL_INDEX(err->mpidr)`; trace `arm_event(err, pei, pei_len, ctx, ctx_len, vsei, vsei_len, sev, cpu)`.

AMD-ATL indirection (`amd_atl_*`):
1. `amd_atl_register_decoder(fn)` sets a `static unsigned long (*)(struct atl_err *)` pointer; the function pointer is intentionally "set once, not unset" except for test/debug; `_unregister_decoder` is provided for unit tests.
2. `amd_convert_umc_mca_addr_to_sys_addr(err)` returns `-EINVAL` if no decoder.

## Hardening

(Inherits from `drivers/ras/00-overview.md`.)

ras-core specific:

- **CEC array size pinned at exactly `PAGE_SIZE`** — single page allocation, no growth, no shrinkage; defeats per-event allocation pressure.
- **PFN binary-search bounded** — `find_elem` is O(log n) on a single page; cannot leak time beyond a single page walk; defeats CPU-load DoS via correctable-error storm.
- **action_threshold poison** — once a PFN's count saturates, `memory_failure` is invoked exactly once; per-PFN entry then deleted; defeats repeated re-poison attempts.
- **`disabled` flag short-circuits everything** — `ras=cec_disable` or runtime debugfs disable is a clean bypass without freeing the array (no UAF on concurrent insert).
- **ARM CPER FW_BUG clamping** — firmware-supplied lengths inconsistent with declared section_length are dump-and-zero rather than overrun — firmware compromise cannot pivot to kernel OOB.
- **AMD-ATL "set once" decoder pointer** — single-writer convention (the `amd-atl` module) plus README warning that unloading is for test only; defeats `amd-atl` swap-out + spoofed-decoder injection.
- **trace ringbuffer mediates exposure** — RAS event details only visible via tracefs (gated by tracing-capable readers) and rasdaemon; not on default dmesg.
- **`extlog_mem_event` tracepoint EXPORT_SYMBOL_GPL** — third-party modules cannot subscribe via in-tree GPL-only export gating.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `atl_err` scratch structs; CEC array is a single `__get_free_page` page tracked as RAS-internal, never USERCOPY-exposed; tracepoint payloads sized exactly per `TP_STRUCT__entry` schema.
- **PAX_KERNEXEC** — RAS core text W^X; trace-event format strings (`TRACE_INCLUDE_PATH ../../include/ras`), `parse_ras_param`, and the AMD-ATL function-pointer slot region live in `__ro_after_init` text where ABI permits.
- **PAX_RANDKSTACK** — randomize kstack offset across `cec_add_elem`, `do_spring_clean`, `log_arm_hw_error`, and `log_non_standard_event`; defeats RAS-storm-driven stack-pivot harvesting.
- **PAX_REFCOUNT** — no per-object refcounts in `drivers/ras/` core, but the indirected `amd_atl_umc_na_to_spa` module reference is `try_module_get`-gated; saturating counter defeats decoder-unload race.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `atl_err` scratch + per-CPER scratch + per-EXT-LOG buffers; correctable-error addresses (which reveal DIMM-layout / DRAM-row patterns) never bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on the debugfs write paths (`pfn`, `count_threshold`, …); user pointer never deref'd outside `kstrtoull` helpers.
- **PAX_RAP / kCFI** — `amd_atl_umc_na_to_spa` indirect call kCFI-typed; tracepoint static-call probe-fn pointers kCFI-typed; debugfs `file_operations` vtable `__ro_after_init`.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-CEC PFN disclosure behind CAP_SYSLOG; `/sys/kernel/debug/ras/array_dump` is debugfs (already root-only) but the leak surface is explicitly noted in `array_dump` show callback.
- **GRKERNSEC_DMESG** — restrict per-MCE / per-AER / per-extlog / per-arm-event kmsg lines (the secondary `pr_err`-style backups complementing the tracepoint) to CAP_SYSLOG; DIMM topology inferable from CECs is a fingerprinting + targeted-rowhammer-aid channel.
- **CEC threshold + debugfs CAP_SYS_ADMIN** — `count_threshold`, `decay_interval`, `pfn`-inject, and `disable` all require CAP_SYS_ADMIN; debugfs root-mode already enforces, but per-file `0600` belt-and-braces.
- **Trace ringbuffer GRKERNSEC_DMESG** — RAS tracepoint payloads carry physical addresses, MPIDRs, and FRU ids that leak hardware topology; tracefs readers must be in the `tracing-capable` group (per existing kernel policy) AND policy-gated CAP_SYSLOG where supported.
- **Error-decoder kCFI** — AMD-ATL `amd_atl_umc_na_to_spa` indirect call is kCFI-typed; any future ARM / Intel address-translation decoder following the same indirection pattern inherits the kCFI requirement.
- **Spring-clean bounded** — `do_spring_clean` walks at most `MAX_ELEMS` u64 entries; defeats unbounded CPU consumption from CEC-storm.
- **ARM-event mpidr→cpu safe-cast** — `cpu = -1` returned cleanly if `GET_LOGICAL_INDEX(mpidr)` fails; downstream tracepoint consumers always see a valid int.
- **AMD-ATL decoder unload guard** — `amd_atl_unregister_decoder` documented as test-only; production policy treats decoder pointer as effectively immutable post-bind.

Rationale: RAS data is paradoxically privileged — it reveals exact physical-address-of-correctable-error history, MPIDR / NUMA-node layout, fuse-revealed silicon-rev, and the operator's DIMM topology. An attacker who can read RAS tracepoints learns which DRAM rows are already stressed (a rowhammer assist) and which DIMM slot a colocated tenant occupies. Conversely, a writer who can spoof the AMD-ATL decoder can mis-attribute every DRAM correctable to wrong DIMMs, hiding a failing module from replacement policy. CAP_SYS_ADMIN gating on debugfs, CAP_SYSLOG on dmesg + tracepoint readers, kCFI on the decoder indirection, and bounded CEC paths turn the RAS subsystem from a "free fingerprint feed" into a controlled disclosure surface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- EDAC core (covered in `drivers/edac/00-overview.md` future Tier-3)
- APEI / GHES firmware-first error handling (covered in `drivers/acpi/apei/` Tier-3)
- ACPI ExtLog (covered in `drivers/acpi/acpi_extlog.md` future Tier-3)
- PCIe AER (covered in `drivers/pci/pcie/aer.md` future Tier-3)
- Per-arch MCE handlers (covered in `arch/x86/kernel/cpu/mce/*` and `arch/arm64/kernel/acpi*.c` Tier-3 if needed)
- amd-atl decoder internals (covered in `drivers/ras/amd/atl/` future Tier-3 `amd-atl.md`)
- `memory_failure` core (covered in `mm/memory-failure.md` Tier-3)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cec_array_no_oob` | OOB | `find_elem`/`del_elem`/`add` operate on `[0, MAX_ELEMS)`; binary-search bounds preserved across `memmove` |
| `cec_count_saturates` | OVERFLOW | per-element count clamped at `COUNT_MASK`; never wraps |
| `arm_cper_lengths_consistent` | RANGE | `pei_len + ctx_len + sizeof(cper_sec_proc_arm) <= err->section_length` enforced before VSEI read |
| `atl_decoder_indirect_kcfi` | TYPE | `amd_atl_umc_na_to_spa` indirect call is kCFI-typed |

### Layer 2: TLA+

`models/ras/cec_lifecycle.tla`: proves a saturating-then-poisoned PFN cannot remain in the CEC array (poison ⇒ del); spring-clean cannot lose un-decayed entries; decay-count + spring-clean threshold reaches `pfns_poisoned` monotonically.

### Layer 3: Verus invariants

- `cec_add_elem` post: `n <= MAX_ELEMS`; for any PFN, at most one entry exists.
- `log_arm_hw_error` post: emits at most one `arm_event` per call; never reads past `err + section_length`.

### Layer 4: Functional

Synthetic CEC storm (`pfn` debugfs inject) drives `pfns_poisoned` increments; `rasdaemon` end-to-end consume on EDAC-bearing test platform; ARM CPER fuzz vs `log_arm_hw_error` (KASAN + UBSAN clean).
