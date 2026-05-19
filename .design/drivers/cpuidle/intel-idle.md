# Tier-3: drivers/idle/intel_idle.c — Intel native idle driver (MWAIT C-states, model-specific tables, package vs core)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/cpuidle/00-overview.md
upstream-paths:
  - drivers/idle/intel_idle.c
  - arch/x86/include/asm/mwait.h
  - arch/x86/include/asm/intel-family.h
  - arch/x86/include/asm/cpuid/api.h
  - Documentation/admin-guide/pm/intel_idle.rst
-->

## Summary

`intel_idle` is the native idle driver for modern Intel processors, replacing the generic ACPI `processor_idle` driver on every Intel part from Nehalem (2008) forward. It exists because Intel CPUs expose richer C-state semantics than ACPI's per-platform _CST tables can describe: per-CPUID model variants, per-stepping latency quirks, MWAIT sub-states (C1, C1E, C3, C6, C7, C8, C9, C10 plus per-state subvariants), IBRS-on-idle plumbing for Spectre-v2 mitigation, large-XSTATE-init on C6 entry, and package-level coordination where multiple cores reaching the same core-C-state may promote to package-C-state. The driver discovers `X86_FEATURE_MWAIT` via CPUID leaf 0x5, reads `mwait_substates`, then selects a model-specific `cpuidle_state[]` table compiled in from one of dozens of per-model arrays (`nhm_cstates`, `snb_cstates`, `hsw_cstates`, `skl_cstates`, `spr_cstates`, `adl_cstates`, etc., plus E-core and atom variants).

This Tier-3 covers `drivers/idle/intel_idle.c` (~2790 lines: per-model tables, MWAIT dispatch, ACPI _CST extract for un-tabled parts, command-line table override, sysfs init).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `intel_idle_driver` (struct cpuidle_driver) | per-system driver instance with intel_idle name | `drivers::intel_idle::Driver` |
| `struct idle_cpu` | per-CPUID-model record (state_table, auto_demotion_disable_flags, disable_promotion_to_c1e, c1_demotion_supported, use_acpi) | `drivers::intel_idle::IdleCpu` |
| `intel_idle_init()` | subsys init: detect CPUID model, pick state_table, register driver, install cpuhp | `Subsystem::init` |
| `intel_idle_cpuidle_driver_init(drv)` | populate `drv->states[]` from `cpuidle_state_table`, skip disabled_states_mask | `Driver::cpuidle_init` |
| `intel_idle_cpu_init(cpu)` | per-CPU init: register cpuidle_device, set `enabled` | `Driver::cpu_init` |
| `intel_idle_cpu_online(cpu)` | cpuhp online callback | `Driver::cpu_online` |
| `intel_idle(dev, drv, idx)` | normal C-state entry: `__intel_idle(dev, drv, idx, true)` (IRQ-disabled MWAIT) | `Driver::enter` |
| `intel_idle_irq(dev, drv, idx)` | IRQ-enabled C-state entry (for `CPUIDLE_FLAG_IRQ_ENABLE` states) | `Driver::enter_irq` |
| `intel_idle_ibrs(dev, drv, idx)` | IBRS-disable around MWAIT then re-enable (KERNEL_IBRS Spectre-v2 path) | `Driver::enter_ibrs` |
| `intel_idle_xstate(dev, drv, idx)` | init large xstate before C6 (Sapphire Rapids quirk) | `Driver::enter_xstate` |
| `intel_idle_s2idle(dev, drv, idx)` | suspend-to-idle deep entry | `Driver::enter_s2idle` |
| `__intel_idle(dev, drv, idx, irqoff)` | core MWAIT issuer: `mwait_idle_with_hints(eax=mwait_hint, ecx=1*irqoff)` | `Driver::__intel_idle` |
| `flg2MWAIT(flags)` / `MWAIT2flg(eax)` | encode MWAIT hint in cpuidle_state.flags top byte | `Driver::flg2mwait` / `_mwait2flg` |
| `intel_idle_acpi_cst_extract()` | fallback: parse ACPI _CST when no model-specific table matches | `Driver::acpi_cst_extract` |
| `cmdline_table_adjust(drv)` | apply `intel_idle.table=` cmdline override (name:latency:residency triples) | `Driver::cmdline_table_adjust` |
| `auto_demotion_disable()` / `c1e_promotion_disable()` | per-model MSR writes to disable HW auto-demotion + C1E promotion | `Driver::auto_demotion_disable` / `_c1e_promotion_disable` |
| `c1_demotion_set(enable)` | set `MSR_PKG_CST_CONFIG_CONTROL.C1_AUTODEMOTE` per `intel_idle.c1_demotion` sysfs | `Driver::c1_demotion_set` |
| Per-model state arrays (`nhm_cstates`, `snb_cstates`, ..., `spr_cstates`, `adl_cstates`) | static const cpuidle_state[] per Intel CPU family | `Driver::*_cstates` |
| Per-model `idle_cpu_*` records + `intel_idle_ids[]` x86_cpu_id MATCH table | CPUID model → idle_cpu record dispatch | `Driver::INTEL_IDLE_IDS` |

## Compatibility contract

REQ-1: CPUID gating: `X86_FEATURE_MWAIT` (leaf 1, ecx bit 3) AND CPUID leaf 5 must report sub-states for at least one C-state. If absent, driver refuses load via `-ENODEV`.

REQ-2: Per-model dispatch: `x86_match_cpu(intel_idle_ids)` returns matching `idle_cpu` record. If unmatched but cmdline `intel_idle.use_acpi=Y` or `force_use_acpi`: fall back to `intel_idle_acpi_cst_extract`.

REQ-3: MWAIT hint encoding: top byte of `cpuidle_state.flags` holds the EAX MWAIT hint (`MWAIT2flg(eax)`); `flg2MWAIT` decodes for `mwait_idle_with_hints(hint, ecx)` call.

REQ-4: ECX bit 0 = "break on interrupt flag" — when 1, MWAIT exits on interrupts even with IF=0 (used by `intel_idle` IRQ-disabled path); when 0, MWAIT requires IF=1 (used by `intel_idle_irq` path).

REQ-5: Per-state flags: `CPUIDLE_FLAG_IRQ_ENABLE` (re-enable IRQs before MWAIT), `CPUIDLE_FLAG_ALWAYS_ENABLE` (enable even if ACPI _CST disagrees), `CPUIDLE_FLAG_IBRS` (KERNEL_IBRS sequence around MWAIT), `CPUIDLE_FLAG_INIT_XSTATE` (init xstate before MWAIT for C6 quirk), `CPUIDLE_FLAG_PARTIAL_HINT_MATCH` (ignore sub-state when matching with ACPI), `CPUIDLE_FLAG_TIMER_STOP` (local APIC stops, need broadcast).

REQ-6: Core vs package C-states: C1, C1E are core-local (per-core MWAIT entry). C3+ on most parts are coordinated: all cores in a package must request the same depth before package-C-state is entered. Per-state target_residency factors package-coordination latency.

REQ-7: Auto-demotion: HW auto-demotion may collapse a requested deep C-state to a shallower one based on package activity. Per-model `auto_demotion_disable_flags` (bitmask of MSR bits in `MSR_NHM_SNB_PKG_CST_CFG_CTL`) cleared at init to defeat demotion when undesirable.

REQ-8: C1E promotion: HW may promote C1 to C1E (deeper). Per-model `disable_promotion_to_c1e` clears `MSR_IA32_POWER_CTL.C1E_PROMOTION` bit.

REQ-9: C1 auto-demote: `MSR_PKG_CST_CONFIG_CONTROL.C1_AUTODEMOTE` toggleable via `/sys/devices/system/cpu/intel_idle/c1_demotion` (CAP_SYS_ADMIN) when `c1_demotion_supported`.

REQ-10: IBRS-off-on-idle: when `X86_FEATURE_KERNEL_IBRS` is set, idle paths must disable IBRS (writing 0 to `MSR_IA32_SPEC_CTRL.IBRS`) to avoid pinning per-core IBRS state and re-enable after MWAIT. `intel_idle_ibrs` implements this; `ibrs_off` cmdline overrides.

REQ-11: XSTATE init: on Sapphire Rapids C6, large AMX xstate must be brought to INIT state before MWAIT to avoid wakeup-latency penalty; `intel_idle_xstate` calls `fpu_idle_fpregs()` before MWAIT.

REQ-12: ACPI _CST integration: when `idle_cpu->use_acpi`, walk ACPI _CST and disable any compiled-in state not also present in _CST (firmware veto), with `CPUIDLE_FLAG_ALWAYS_ENABLE` overriding the veto.

REQ-13: Cmdline overrides: `intel_idle.max_cstate=N` caps deepest selectable index; `intel_idle.states_off=BITMASK` disables specific state indices; `intel_idle.table=name:latency:residency,...` overrides table at boot.

## Acceptance Criteria

- [ ] AC-1: `ls /sys/devices/system/cpu/cpu0/cpuidle/` shows `state0`..`state6` (or per-model count) on Intel reference HW; `state*/name` shows POLL, C1, C1E, C3, C6, C7s, C8, C9, C10 per part.
- [ ] AC-2: `dmesg | grep intel_idle` shows MWAIT-substates table + per-state init banner.
- [ ] AC-3: Per-state usage counters increment under idle workload; `time` matches wall-clock idle.
- [ ] AC-4: turbostat reports core + package C-state residency consistent with cpuidle counters.
- [ ] AC-5: Cmdline test: `intel_idle.max_cstate=1` limits driver to POLL + C1; deeper states absent from sysfs.
- [ ] AC-6: `intel_idle.states_off=0x4` disables state index 2 (C3 typically); sysfs `disable` = 1.
- [ ] AC-7: ibrs_off cmdline test: with KERNEL_IBRS, `intel_idle.ibrs_off=Y` reverts to plain `intel_idle` enter (no IBRS toggle).
- [ ] AC-8: c1_demotion sysfs test (supported parts): toggle changes `MSR_PKG_CST_CONFIG_CONTROL` bit observable via `rdmsr 0xe2`.
- [ ] AC-9: ACPI fallback test: boot Intel part with no model-specific table (or `intel_idle.use_acpi=Y`); _CST-derived states appear.

## Architecture

`Driver` (struct cpuidle_driver `intel_idle_driver`) is populated from a per-model `cpuidle_state[]` table at `intel_idle_init`:

```
struct idle_cpu {
  state_table: *const [CpuIdleState],
  auto_demotion_disable_flags: u64,    // MSR bits to clear
  disable_promotion_to_c1e: bool,
  c1_demotion_supported: bool,
  use_acpi: bool,
}

// Example per-model:
const SKL_CSTATES: &[CpuIdleState] = &[
  // state[0] is POLL inserted by core
  CpuIdleState { name: "C1", desc: "MWAIT 0x00", flags: MWAIT2flg(0x00), exit_latency_ns: 2000, target_residency_ns: 2000, enter: intel_idle },
  CpuIdleState { name: "C1E", desc: "MWAIT 0x01", flags: MWAIT2flg(0x01) | CPUIDLE_FLAG_ALWAYS_ENABLE, exit_latency_ns: 10000, target_residency_ns: 20000, enter: intel_idle },
  CpuIdleState { name: "C3", desc: "MWAIT 0x10", flags: MWAIT2flg(0x10) | CPUIDLE_FLAG_TIMER_STOP, exit_latency_ns: 70000, target_residency_ns: 100000, enter: intel_idle },
  CpuIdleState { name: "C6", desc: "MWAIT 0x20", flags: MWAIT2flg(0x20) | CPUIDLE_FLAG_TIMER_STOP, exit_latency_ns: 85000, target_residency_ns: 200000, enter: intel_idle },
  CpuIdleState { name: "C7s", desc: "MWAIT 0x33", flags: MWAIT2flg(0x33) | CPUIDLE_FLAG_TIMER_STOP, exit_latency_ns: 124000, target_residency_ns: 800000, enter: intel_idle },
  CpuIdleState { name: "C8", desc: "MWAIT 0x40", flags: MWAIT2flg(0x40) | CPUIDLE_FLAG_TIMER_STOP, exit_latency_ns: 200000, target_residency_ns: 800000, enter: intel_idle },
  // ... C9, C10 ...
];

static const X86_MATCH_INTEL_FAM6_MODEL_KABYLAKE: x86_cpu_id = { ..., driver_data: &idle_cpu_skl };
```

Subsystem init `intel_idle_init`:
1. `cpu_feature_enabled(X86_FEATURE_MWAIT)` check → fail otherwise.
2. `cpuid(0x05, &eax, &ebx, &ecx, &edx); mwait_substates = edx;` — sub-state inventory.
3. `id = x86_match_cpu(intel_idle_ids)` — per-CPUID dispatch.
4. If `id`: `icpu = id->driver_data; cpuidle_state_table = icpu->state_table; auto_demotion_disable_flags = icpu->auto_demotion_disable_flags;` etc.
5. If unmatched and not `force_use_acpi`: fail with `-ENODEV`.
6. `intel_idle_cpuidle_devices = alloc_percpu(struct cpuidle_device)`.
7. `intel_idle_cpuidle_driver_init(&intel_idle_driver)` — copy `cpuidle_state_table` into `intel_idle_driver.states[]`, skip per-`disabled_states_mask`, apply IRQ_ENABLE/IBRS variant selection.
8. `cmdline_table_adjust(&intel_idle_driver)` — apply `intel_idle.table=` overrides.
9. `intel_idle_sysfs_init()` — create `/sys/devices/system/cpu/intel_idle/`.
10. `cpuidle_register_driver(&intel_idle_driver)`.
11. `cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, "idle/intel:online", intel_idle_cpu_online, NULL)` — per-CPU online registration.

Per-CPU online `intel_idle_cpu_online`:
1. `intel_idle_cpu_init(cpu)` — per-CPU state init.
2. If `c1e_promotion == DISABLE`: `c1e_promotion_disable()` (write `MSR_IA32_POWER_CTL.C1E_PROMOTION = 0`).
3. If `auto_demotion_disable_flags != 0`: clear those bits in `MSR_NHM_SNB_PKG_CST_CFG_CTL`.

Idle entry `__intel_idle`:
1. `state = &drv->states[index]; eax = flg2MWAIT(state->flags); ecx = 1*irqoff` (break-on-IRQ flag).
2. `mwait_idle_with_hints(eax, ecx)` — issues `monitor` on a dummy line + `mwait(eax, ecx)`.
3. Return `index` (caller computes residency).

IRQ-enabled variant `intel_idle_irq`:
1. `raw_local_irq_enable()` before MWAIT.
2. `__intel_idle(dev, drv, idx, false)` — ecx=0, MWAIT only exits on IF=1+IRQ.
3. On return, IRQs already disabled by MWAIT-exit hardware contract.

IBRS-off variant `intel_idle_ibrs`:
1. `spec_ctrl_current() & ~SPEC_CTRL_IBRS` → write `MSR_IA32_SPEC_CTRL` to clear IBRS.
2. `__intel_idle(dev, drv, idx, true)`.
3. `wrmsrl(MSR_IA32_SPEC_CTRL, spec_ctrl_current())` — re-enable.

XSTATE init variant `intel_idle_xstate`:
1. `fpu_idle_fpregs()` — bring AMX/AVX-512 xstate to INIT mask via XRSTOR.
2. `__intel_idle(dev, drv, idx, true)`.

ACPI fallback `intel_idle_acpi_cst_extract`:
1. Walk ACPI _CST for boot CPU.
2. Per-_CST entry: if `Type == ACPI_TYPE_FFH` (MWAIT extension), extract `Address` (MWAIT hint) + `AccessSize` (sub-state).
3. Match against `cpuidle_state_table` via MWAIT hint (possibly with PARTIAL_HINT_MATCH); for un-matched parts build a fresh table from _CST.

## Hardening

(Inherits row-1 features from `drivers/cpuidle/00-overview.md` § Hardening.)

intel_idle specific reinforcement:

- **Per-model state_table compiled `const`** — defense against runtime tampering of MWAIT hints.
- **MWAIT hint range checked 0-255** — defense against malformed cmdline table injecting invalid hint.
- **`mwait_idle_with_hints` issues `monitor` on a known-safe address** — defense against spurious wakeup via cache-line poisoning.
- **IRQ-disabled MWAIT (ecx bit 0 = 1) for non-`CPUIDLE_FLAG_IRQ_ENABLE` states** — defense against MWAIT-with-IF=1 race that loses pending IRQ.
- **IBRS toggle requires kernel-IBRS feature** — defense against IBRS write on non-IBRS HW yielding GP fault.
- **Per-CPU `cpuidle_device` alloc_percpu** — fixed at init; no late realloc.
- **CPUHP_AP_ONLINE_DYN registration paired with unregister** — defense against per-CPU state leak on driver removal.
- **`max_cstate` clamp** — defense against `intel_idle.max_cstate=999` overshooting `CPUIDLE_STATE_MAX`.
- **Cmdline table length bounded by `MAX_CMDLINE_TABLE_LEN`** — defense against overlong table parse.
- **Cmdline per-entry latency / residency bounded (5ms / 100ms)** — defense against user-supplied huge values causing governor mis-prediction.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `cpuidle_device` per-CPU arena and ACPI _CST extraction buffers; cmdline-table parser bounded by `MAX_CMDLINE_TABLE_LEN`.
- **PAX_KERNEXEC** — intel_idle driver core in W^X kernel text; per-model `cpuidle_state[]` arrays and `idle_cpu` records compiled `const` + linked into `.rodata`; `intel_idle_driver.states[]` populated at init then frozen `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `intel_idle`, `intel_idle_irq`, `intel_idle_ibrs`, `intel_idle_xstate`, and `intel_idle_s2idle` entry points.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-CPU cpuidle_device pinning across CPU hotplug; overflow trap defeats hotplug + driver-unregister race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for cmdline-table parse staging, per-CPU device structs on hotplug-offline, and ACPI _CST extraction temporaries; prevents leaked MWAIT-hint copies and per-CPU residency residuals.
- **PAX_UDEREF** — SMAP/PAN enforced on every `intel_idle` sysfs store (`c1_demotion`, `max_cstate`, `states_off`); reject user-pointer deref outside `kstrtouint_from_user`.
- **PAX_RAP / kCFI** — `intel_idle_driver.states[].enter` (variants: plain, irq, ibrs, xstate, s2idle) marked `__ro_after_init` with kCFI-typed indirect dispatch; per-model `idle_cpu` records resolved through const tables.
- **GRKERNSEC_HIDESYM** — gate kallsyms for intel_idle helpers and `intel_idle_cpuidle_devices` percpu pointer behind CAP_SYSLOG; suppress `%p` in `intel_idle` enter/exit tracepoints.
- **GRKERNSEC_DMESG** — restrict intel_idle init banners, per-model sub-state inventory, and ACPI _CST extraction diagnostics to CAP_SYSLOG so attackers cannot fingerprint silicon stepping via dmesg.
- **MWAIT hint table RO** — per-model `cpuidle_state[]` table declared `static const` in `.rodata` + W^X-enforced; refuse runtime mutation that would let a sysfs/debugfs write redirect MWAIT hints to attacker-chosen sub-states (e.g., a partial-RMW into `state->flags` to corrupt `flg2MWAIT(state->flags)`).
- **Model-specific quirks RAP** — auto_demotion_disable + c1e_promotion + ibrs / xstate / irq variant dispatch resolved through const idle_cpu records with kCFI-typed callsites; defense against function-pointer overwrite redirecting C6 entry to an attacker-controlled path.
- **C6 demotion gate** — `c1_demotion` sysfs store CAP_SYS_ADMIN + audit; defense against unprivileged enable of C1 auto-demote that would silently break realtime latency assumptions.
- **Cmdline table parser audit** — `intel_idle.table=` parse result logged with caller (boot context) and rejected entries enumerated; defense against post-boot taint via `/proc/cmdline`-style spoof.
- **IBRS toggle paired** — `intel_idle_ibrs` pre-MWAIT write and post-MWAIT re-write paired in the same function; refuse partial sequences that would leave IBRS disabled on return to ring-0.

Rationale: intel_idle owns the literal silicon sleep transition — every MWAIT hint, every IBRS toggle, every package-C-state coordination point is a place where a corrupted state-table or a hijacked enter pointer can either leak microarchitectural state across sleep (skipped IBRS, skipped XSTATE INIT) or wedge the package below the deepest safe C-state. RAP/kCFI on every `.enter` variant, `__ro_after_init` per-model state tables, audit on demotion and cmdline-table mutations, refcount overflow trapping on per-CPU pinning, and explicit IBRS-pairing checks turn intel_idle from "trust the per-model table" into a structural enforcement boundary that meets both the silicon's MWAIT-sub-state spec and the Spectre-v2 mitigation contract.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Generic ACPI `processor_idle` driver (covered in `acpi-processor-idle.md` future Tier-3)
- AMD idle paths (covered in `amd-idle.md` future Tier-3)
- Per-state `enter_dead` deep-park sequences (covered in CPU offline Tier-3 future)
- intel_idle perfmon integration (covered in `intel-perfmon.md` future Tier-3)
- Package vs Core C-state coordination details (covered in `power-coord.md` future Tier-3)
- 32-bit-only paths
- Implementation code
