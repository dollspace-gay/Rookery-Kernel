# Tier-3: arch/x86/kernel/apic/apic.c — x86 Local APIC

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: arch/x86/00-overview.md
upstream-paths:
  - arch/x86/kernel/apic/apic.c (~2699 lines)
  - arch/x86/include/asm/apic.h
  - arch/x86/include/asm/apicdef.h
  - arch/x86/include/asm/msr-index.h (MSR_IA32_APICBASE, MSR_IA32_TSC_DEADLINE)
  - arch/x86/include/asm/idtentry.h (sysvec_apic_timer_interrupt et al.)
-->

## Summary

The Local APIC (LAPIC) is the per-CPU interrupt controller programmed via either a 4 KB MMIO window at the address in `MSR_IA32_APICBASE` (xAPIC mode) or via the MSR window starting at `APIC_BASE_MSR = 0x800` (x2APIC mode). `arch/x86/kernel/apic/apic.c` owns: per-CPU LAPIC enable/disable, **xAPIC ↔ x2APIC mode switching**, **MMIO ↔ MSR register-access dispatch** (`native_apic_msr_read`/`_write` + `apic_read`/`apic_write`), **`setup_local_APIC`** (per-CPU bringup: TPR, SPIV, LVT0/1, ESR, ISR drain), **`clear_local_APIC`** + **`disable_local_APIC`** + **`apic_soft_disable`** (shutdown / kexec / CPU-hotplug paths), the **LAPIC timer clockevent** (oneshot, periodic, and TSC-deadline modes via `MSR_IA32_TSC_DEADLINE`), the **calibration loop** (`calibrate_APIC_clock` using PIT or PM-timer), **NMI delivery** (LINT1 ExtNMI mode: BSP/ALL/NONE), **EILVT slot allocation** (AMD extended LVT for MCE-threshold / IBS), the **spurious-interrupt** and **APIC error** handlers, **PM suspend/resume** (`lapic_suspend`/`lapic_resume`), **`init_apic_mappings`** (LAPIC fixmap), and **`apic_bsp_setup`** (the BSP's symmetric-I/O-mode entry point). Per-`apic_intr_mode_select` picks one of `APIC_PIC`, `APIC_VIRTUAL_WIRE`, `APIC_VIRTUAL_WIRE_NO_CONFIG`, `APIC_SYMMETRIC_IO`, `APIC_SYMMETRIC_IO_NO_ROUTING` based on cmdline + BIOS state + MP/MADT. Critical for: every interrupt, every IPI, every timer tick.

This Tier-3 covers `arch/x86/kernel/apic/apic.c` (~2699 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `apic_mmio_base` (singleton) | per-LAPIC MMIO base | `Apic::MMIO_BASE` |
| `mp_lapic_addr` (singleton) | per-MADT-reported phys addr | `Apic::PHYS_ADDR` |
| `boot_cpu_physical_apicid` | per-BSP APIC ID | shared |
| `boot_cpu_apic_version` | per-BSP APIC version | shared |
| `apic_is_disabled` | per-`nolapic` / BIOS-disable | shared |
| `x2apic_mode` / `x2apic_state` | per-x2APIC enable + lock state | shared |
| `apic_intr_mode` (enum) | per-mode select | shared |
| `apic_extnmi` | per-NMI delivery (BSP/ALL/NONE) | shared |
| `lapic_timer_period` | per-calibration result | shared |
| `lapic_clockevent` | per-CPU clock-event template | `Apic::CLOCKEVENT_TEMPLATE` |
| `lapic_events` (per_cpu) | per-CPU live clockevent | `Apic::EVENTS` |
| `native_apic_icr_write()` / `_icr_read()` | per-ICR low+high | `Apic::{icr_write, icr_read}` |
| `native_apic_msr_read()` / `_write()` | per-x2APIC register access | `Apic::{msr_read, msr_write}` |
| `native_apic_msr_eoi()` | per-x2APIC EOI | `Apic::msr_eoi` |
| `native_x2apic_icr_write()` / `_read()` | per-x2APIC ICR (one 64-bit MSR) | `Apic::{x2apic_icr_write, x2apic_icr_read}` |
| `lapic_get_maxlvt()` | per-LVR LVT-entry count | `Apic::get_maxlvt` |
| `lapic_get_version()` | per-LVR version | `Apic::get_version` |
| `lapic_is_integrated()` | per-integrated-vs-82489DX | `Apic::is_integrated` |
| `setup_APIC_eilvt()` | per-AMD EILVT slot assign | `Apic::setup_eilvt` |
| `reserve_eilvt_offset()` | per-EILVT cmpxchg-reserve | `Apic::reserve_eilvt_offset` |
| `__setup_APIC_LVTT()` | per-LVTT program | `Apic::setup_lvtt` |
| `lapic_next_event()` | per-oneshot APIC_TMICT write | `Apic::next_event` |
| `lapic_next_deadline()` | per-TSC-deadline MSR write | `Apic::next_deadline` |
| `lapic_timer_shutdown()` | per-LVTT mask + TMICT zero | `Apic::timer_shutdown` |
| `lapic_timer_set_periodic()` / `_set_oneshot()` | per-LVTT mode set | `Apic::{timer_set_periodic, timer_set_oneshot}` |
| `lapic_timer_broadcast()` | per-IPI-mask broadcast | `Apic::timer_broadcast` |
| `setup_APIC_timer()` | per-CPU register clockevent | `Apic::setup_timer` |
| `lapic_update_tsc_freq()` | per-recalibration TSC update | `Apic::update_tsc_freq` |
| `apic_validate_deadline_timer()` | per-microcode-rev check | `Apic::validate_deadline_timer` |
| `calibrate_APIC_clock()` | per-PIT/PMTMR/HPET calibration | `Apic::calibrate_clock` |
| `lapic_cal_handler()` | per-calibration tick handler | `Apic::cal_handler` |
| `setup_boot_APIC_clock()` | per-BSP timer setup | `Apic::setup_boot_clock` |
| `setup_secondary_APIC_clock()` | per-AP timer setup | `Apic::setup_secondary_clock` |
| `apic_needs_pit()` | per-PIT-still-needed predicate | `Apic::needs_pit` |
| `local_apic_timer_interrupt()` | per-vector handler body | `Apic::timer_interrupt_body` |
| `sysvec_apic_timer_interrupt` | per-IDT entry | `Apic::timer_idtentry` |
| `clear_local_APIC()` | per-mask all LVT entries | `Apic::clear_local` |
| `apic_soft_disable()` | per-clear + soft-disable in SPIV | `Apic::soft_disable` |
| `disable_local_APIC()` | per-soft-disable + (x86_32) APICBASE off | `Apic::disable_local` |
| `lapic_shutdown()` | per-IRQ-off + disable | `Apic::shutdown` |
| `sync_Arb_IDs()` | per-arbitration-ID-sync IPI | `Apic::sync_arb_ids` |
| `apic_intr_mode_select()` | per-mode pick | `Apic::intr_mode_select` |
| `apic_intr_mode_init()` | per-mode commit | `Apic::intr_mode_init` |
| `init_bsp_APIC()` | per-virtual-wire setup | `Apic::init_bsp` |
| `lapic_setup_esr()` | per-Error-Status-Register init | `Apic::setup_esr` |
| `apic_clear_isr()` / `apic_check_and_eoi_isr()` | per-stale-ISR drain | `Apic::{clear_isr, check_and_eoi_isr}` |
| `setup_local_APIC()` | per-CPU LAPIC bringup | `Apic::setup_local` |
| `end_local_APIC_setup()` | per-ESR setup + PM-activate | `Apic::end_local_setup` |
| `apic_ap_setup()` | per-AP entry from smpboot | `Apic::ap_setup` |
| `apic_read_boot_cpu_id()` | per-BSP APIC ID + version snapshot | `Apic::read_boot_cpu_id` |
| `check_x2apic()` | per-BIOS x2APIC-state probe | `Apic::check_x2apic` |
| `x2apic_hw_locked()` | per-`MSR_IA32_XAPIC_DISABLE_STATUS` check | `Apic::x2apic_hw_locked` |
| `__x2apic_enable()` / `__x2apic_disable()` | per-APICBASE.X2APIC_ENABLE | `Apic::{x2apic_enable_msr, x2apic_disable_msr}` |
| `x2apic_setup()` | per-AP x2APIC enable | `Apic::x2apic_setup` |
| `try_to_enable_x2apic()` | per-IR-aware enable | `Apic::try_to_enable_x2apic` |
| `enable_IR_x2apic()` | per-interrupt-remap + x2APIC | `Apic::enable_ir_x2apic` |
| `detect_init_APIC()` | per-LAPIC presence probe | `Apic::detect_init` |
| `apic_force_enable()` | per-APICBASE force re-enable (x86_32) | `Apic::force_enable` |
| `apic_verify()` | per-CPUID + APICBASE-base read | `Apic::verify` |
| `init_apic_mappings()` | per-LAPIC fixmap | `Apic::init_mappings` |
| `apic_set_fixmap()` | per-FIX_APIC_BASE pre-set | `Apic::set_fixmap` |
| `register_lapic_address()` | per-MADT-reported addr commit | `Apic::register_address` |
| `handle_spurious_interrupt()` | per-vector spurious handler | `Apic::handle_spurious` |
| `sysvec_spurious_apic_interrupt` / `sysvec_error_interrupt` | per-IDT entry | `Apic::{spurious_idtentry, error_idtentry}` |
| `connect_bsp_APIC()` / `disconnect_bsp_APIC()` | per-IMCR mode switch (x86_32) | `Apic::{connect_bsp, disconnect_bsp}` |
| `__irq_msi_compose_msg()` | per-MSI address/data fill | `Apic::compose_msi_msg` |
| `x86_msi_msg_get_destid()` | per-MSI dest-id decode | `Apic::msi_get_destid` |
| `apic_bsp_setup()` | per-BSP symmetric-I/O setup | `Apic::bsp_setup` |
| `apic_bsp_up_setup()` | per-UP-mode BSP setup | `Apic::bsp_up_setup` |
| `lapic_suspend()` / `lapic_resume()` | per-PM save/restore | `Apic::{suspend, resume}` |
| `apic_pm_activate()` | per-PM activate flag | `Apic::pm_activate` |
| `apic_is_clustered_box()` | per-DMI multi-chassis detect | `Apic::is_clustered_box` |
| `lapic_insert_resource()` | per-late iomem insert | `Apic::insert_resource` |

## Compatibility contract

REQ-1: APIC register layout (xAPIC MMIO at `apic_mmio_base`, x2APIC MSRs at `APIC_BASE_MSR + (reg >> 4)`):
- `APIC_ID` (0x20), `APIC_LVR` (0x30, version), `APIC_TASKPRI` (0x80, TPR), `APIC_EOI` (0xB0),
  `APIC_LDR` (0xD0), `APIC_DFR` (0xE0), `APIC_SPIV` (0xF0, spurious-vector + APIC-enable),
  `APIC_ISR` (0x100, 8×32-bit), `APIC_TMR` (0x180), `APIC_IRR` (0x200), `APIC_ESR` (0x280),
  `APIC_LVTCMCI` (0x2F0), `APIC_ICR` (0x300, ICR-low), `APIC_ICR2` (0x310, ICR-high),
  `APIC_LVTT` (0x320, timer LVT), `APIC_LVTTHMR` (0x330, thermal LVT), `APIC_LVTPC` (0x340, PMC LVT),
  `APIC_LVT0` (0x350), `APIC_LVT1` (0x360, NMI LVT), `APIC_LVTERR` (0x370),
  `APIC_TMICT` (0x380, timer initial count), `APIC_TMCCT` (0x390, current count),
  `APIC_TDCR` (0x3E0, divide config).

REQ-2: native_apic_msr_write(reg, v):
- if `reg` ∈ {`APIC_DFR`, `APIC_ID`, `APIC_LDR`, `APIC_LVR`}: silent return (read-only in x2APIC).
- else: `wrmsrq(APIC_BASE_MSR + (reg >> 4), v)`.

REQ-3: native_apic_msr_read(reg) -> u32:
- if `reg == APIC_DFR`: return `(u32)-1` (DFR doesn't exist in x2APIC).
- else: `rdmsrq(APIC_BASE_MSR + (reg >> 4), &msr)`; return `(u32)msr`.

REQ-4: native_apic_msr_eoi():
- `native_wrmsrq(APIC_BASE_MSR + (APIC_EOI >> 4), APIC_EOI_ACK)`.

REQ-5: native_apic_icr_write(low, id):
- xAPIC: irq-saved sequence:
  - `apic_write(APIC_ICR2, SET_XAPIC_DEST_FIELD(id))`.
  - `apic_write(APIC_ICR, low)`.
- x2APIC: `wrmsrq(APIC_BASE_MSR + (APIC_ICR >> 4), ((u64)id << 32) | low)` (single 64-bit write).

REQ-6: setup_local_APIC (per-CPU bringup, preemption disabled):
1. if `apic_is_disabled`: `disable_ioapic_support`; return.
2. `apic->setup()` (driver hook).
3. Soft-disable: clear `APIC_SPIV_APIC_ENABLED` in SPIV (kexec/kcrash leftover-safety).
4. (x86_32 + integrated + `apic->disable_esr`) write `APIC_ESR=0` four times.
5. `apic->init_apic_ldr()` (driver hook) for logical-destination mode.
6. TPR = `(read & ~APIC_TPRI_MASK) | 0x10` (accept everything except vectors 0–31).
7. `apic_clear_isr()` (drain stale ISR + IRR — REQ-13).
8. SPIV: clear `APIC_VECTOR_MASK`, set `APIC_SPIV_APIC_ENABLED`, set vector to `SPURIOUS_APIC_VECTOR` (0xff), clear `APIC_SPIV_FOCUS_DISABLED` (x86_32).
9. `perf_events_lapic_init()`.
10. LVT0:
    - if BSP **and** (`pic_mode` || `LVT0 unmasked` || `ioapic_is_disabled`): `APIC_DM_EXTINT`.
    - else: `APIC_DM_EXTINT | APIC_LVT_MASKED`.
11. LVT1:
    - if (BSP **and** `apic_extnmi != NONE`) **or** `apic_extnmi == ALL`: `APIC_DM_NMI`.
    - else: `APIC_DM_NMI | APIC_LVT_MASKED`.
    - if non-integrated (82489DX): OR-in `APIC_LVT_LEVEL_TRIGGER`.
12. (CONFIG_X86_MCE_INTEL && BSP) `cmci_recheck()`.

REQ-7: clear_local_APIC:
1. Return if `!apic_accessible()` (neither x2APIC nor MMIO mapped).
2. Compute `maxlvt = lapic_get_maxlvt()`.
3. If `maxlvt >= 3`: write `APIC_LVTERR = ERROR_APIC_VECTOR | APIC_LVT_MASKED` **first** (otherwise masking with vector==0 can trigger a local APIC error).
4. Mask LVTT, LVT0, LVT1 (and LVTPC if `maxlvt >= 4`).
5. Mask LVTTHMR if `maxlvt >= 5` (CONFIG_X86_THERMAL_VECTOR).
6. Mask LVTCMCI if `maxlvt >= 6` and currently unmasked (CONFIG_X86_MCE_INTEL).
7. Clean state: write `APIC_LVT_MASKED` to LVTT, LVT0, LVT1, LVTERR, LVTPC.
8. If integrated and `maxlvt > 3`: write `APIC_ESR = 0` (Pentium errata 3AP/11AP), then read `APIC_ESR` to clear.

REQ-8: apic_soft_disable:
1. `clear_local_APIC()`.
2. SPIV: clear `APIC_SPIV_APIC_ENABLED`; write back.

REQ-9: disable_local_APIC:
1. Return if `!apic_accessible()`.
2. `apic->teardown()` if non-NULL.
3. `apic_soft_disable()`.
4. (x86_32 + `enabled_via_apicbase`) clear `MSR_IA32_APICBASE_ENABLE` bit in `MSR_IA32_APICBASE`.

REQ-10: __setup_APIC_LVTT(clocks, oneshot, irqen):
1. lvtt = `LOCAL_TIMER_VECTOR`.
2. if not oneshot: OR `APIC_LVT_TIMER_PERIODIC`.
3. else if `X86_FEATURE_TSC_DEADLINE_TIMER`: OR `APIC_LVT_TIMER_TSCDEADLINE`.
4. if not integrated (82489DX): OR `I82489DX_BASE_DIVIDER = (0x2 << 18)`.
5. if not irqen: OR `APIC_LVT_MASKED`.
6. `apic_write(APIC_LVTT, lvtt)`.
7. if TSC-deadline bit set: `mfence` and return (no TDCR/TMICT in TSC-deadline mode).
8. Else: `apic_write(APIC_TDCR, (read & ~(DIV_1|DIV_TMBASE)) | DIV_16)`.
9. if not oneshot: `apic_write(APIC_TMICT, clocks / APIC_DIVISOR)` where `APIC_DIVISOR = 16`.

REQ-11: lapic_next_event(delta, evt) (oneshot, non-TSC-deadline):
1. `apic_write(APIC_TMICT, delta)`; return 0.

REQ-12: lapic_next_deadline(delta, evt) (TSC-deadline mode):
1. `tsc = rdtsc()`.
2. `native_wrmsrq(MSR_IA32_TSC_DEADLINE, tsc + (u64)delta * TSC_DIVISOR)` where `TSC_DIVISOR = 8`.
3. return 0.

REQ-13: apic_clear_isr + apic_check_and_eoi_isr:
- `apic_check_and_eoi_isr(isr)`:
  - Read 8 `APIC_ISR + i*0x10` registers into `isr.regs[]` (256 bits).
  - If bitmap_empty: return true.
  - For each set bit: `apic_eoi()`.
  - Reread ISR; return bitmap_empty.
- `apic_clear_isr()`:
  - If `!apic_check_and_eoi_isr(&ir)`: `pr_warn("APIC: Stale ISR: %256pb\n", ir.map)`.
  - Read IRR; if non-empty: `pr_warn("APIC: Stale IRR: %256pb\n", ir.map)`.

REQ-14: lapic_timer_shutdown(evt):
1. If `evt.features & CLOCK_EVT_FEAT_DUMMY`: return 0.
2. `v = apic_read(APIC_LVTT) | APIC_LVT_MASKED | LOCAL_TIMER_VECTOR`.
3. `apic_write(APIC_LVTT, v)`.
4. If `v & APIC_LVT_TIMER_TSCDEADLINE`: `native_wrmsrq(MSR_IA32_TSC_DEADLINE, 0)`.
5. Else: `apic_write(APIC_TMICT, 0)`.

REQ-15: setup_APIC_timer (per-CPU clockevent register):
1. `levt = this_cpu_ptr(&lapic_events)`.
2. If `X86_FEATURE_ARAT`: clear `CLOCK_EVT_FEAT_C3STOP` on template; bump rating to 150.
3. `memcpy(levt, &lapic_clockevent, sizeof(*levt))`.
4. `levt->cpumask = cpumask_of(smp_processor_id())`.
5. If `X86_FEATURE_TSC_DEADLINE_TIMER`:
   - name = "lapic-deadline".
   - Clear PERIODIC + DUMMY; set `CLOCK_EVT_FEAT_CLOCKSOURCE_COUPLED`.
   - `cs_id = CSID_X86_TSC`.
   - `set_next_event = lapic_next_deadline`.
   - `clockevents_config_and_register(levt, tsc_khz * 1000 / TSC_DIVISOR, 0xF, ~0UL)`.
6. Else: `clockevents_register_device(levt)`.
7. `apic_update_vector(smp_processor_id(), LOCAL_TIMER_VECTOR, true)`.

REQ-16: apic_intr_mode select (__apic_intr_mode_select):
- If `apic_is_disabled`: `APIC_PIC`.
- x86_64: if `!X86_FEATURE_APIC`: set `apic_is_disabled = true`; `APIC_PIC`.
- x86_32: if neither integrated APIC nor SMP found and not 82489DX: `APIC_PIC`; if BIOS lies about integrated APIC: `APIC_PIC`.
- If `!smp_found_config`: `disable_ioapic_support()`; if `!acpi_lapic`: `APIC_VIRTUAL_WIRE_NO_CONFIG`; else: `APIC_VIRTUAL_WIRE`.
- (CONFIG_SMP) if `!setup_max_cpus`: `APIC_SYMMETRIC_IO_NO_ROUTING`.
- Else: `APIC_SYMMETRIC_IO`.

REQ-17: x2APIC state machine (`x2apic_state`):
- `X2APIC_OFF` → initial.
- `X2APIC_DISABLED` → cmdline `nox2apic` or no `X86_FEATURE_X2APIC`.
- `X2APIC_ON` → BIOS-enabled or kernel-enabled.
- `X2APIC_ON_LOCKED` → hardware-locked via `MSR_IA32_XAPIC_DISABLE_STATUS` with `LEGACY_XAPIC_DISABLED` set.
- Transitions:
  - `check_x2apic()` reads BIOS state.
  - `try_to_enable_x2apic(remap_mode)`: if `remap_mode != IRQ_REMAP_X2APIC_MODE` and no IR hypervisor support: `x2apic_disable()`; else: `x2apic_enable()`.
  - `x2apic_setup()` (per-AP): match BSP state; if `< X2APIC_ON`: `__x2apic_disable`; else `__x2apic_enable`.

REQ-18: __x2apic_enable / __x2apic_disable (`MSR_IA32_APICBASE = 0x1b`):
- Bits: `MSR_IA32_APICBASE_BSP (8)`, `X2APIC_ENABLE (10)`, `XAPIC_ENABLE (11)`.
- `__x2apic_enable`: read APICBASE; if `X2APIC_ENABLE` set: return. Else write APICBASE | `X2APIC_ENABLE`.
- `__x2apic_disable`: read; if not enabled: return. Clear both `X2APIC_ENABLE | XAPIC_ENABLE` first, then re-enable only `XAPIC_ENABLE`.

REQ-19: setup_APIC_eilvt(offset, vector, msg_type, mask) (AMD extended-LVT slots):
- `new = (mask << 16) | (msg_type << 8) | vector`.
- `reg = APIC_EILVTn(offset)` (offset 0–3, registers 0x500/0x510/0x520/0x530).
- `reserved = reserve_eilvt_offset(offset, new)`:
  - Atomic cmpxchg on `eilvt_offsets[offset]` until vector slot agreed; `~0` if `offset >= APIC_EILVT_NR_MAX = 4`; existing-different-vector returns the existing reservation.
- If `reserved != new`: `pr_err FW_BUG; return -EINVAL`.
- If existing vector not changeable: `pr_err FW_BUG; return -EBUSY`.
- `apic_write(reg, new)`; return 0.

REQ-20: LVT-NMI delivery (apic_extnmi):
- `APIC_EXTNMI_BSP` (default): only BSP LINT1 unmasked NMI.
- `APIC_EXTNMI_ALL`: every CPU LINT1 unmasked NMI.
- `APIC_EXTNMI_NONE`: all LINT1 masked.
- Parsed from `apic_extnmi={bsp|all|none}` cmdline.

REQ-21: CPU-hotplug APIC-on/off:
- AP-bringup: `apic_ap_setup()` = `setup_local_APIC()` + `end_local_APIC_setup()`.
- AP-bringdown: `apic_soft_disable()` (REQ-8); APICBASE remains set; AP can be re-launched via INIT/SIPI.
- x2APIC AP enable: `x2apic_setup()` (REQ-17 transition).

REQ-22: sysvec_apic_timer_interrupt (IDT entry, vector `LOCAL_TIMER_VECTOR`):
1. `old_regs = set_irq_regs(regs)`.
2. `apic_eoi()`.
3. `trace_local_timer_entry(LOCAL_TIMER_VECTOR)`.
4. `local_apic_timer_interrupt()`:
   - If `evt.event_handler == NULL`: `pr_warn("Spurious LAPIC timer interrupt on cpu %d")` + `lapic_timer_shutdown(evt)`; return.
   - `inc_irq_stat(apic_timer_irqs)`.
   - `evt.event_handler(evt)`.
5. `trace_local_timer_exit(LOCAL_TIMER_VECTOR)`.
6. `set_irq_regs(old_regs)`.

REQ-23: sysvec_spurious_apic_interrupt / sysvec_error_interrupt:
- Spurious (vector 0xff): bump `irq_spurious_count`; if vector == `SPURIOUS_APIC_VECTOR`: pr_info "should never happen"; else verify ISR bit + `apic_eoi`.
- Error: write `APIC_ESR = 0` (Pentium errata 3AP); read `APIC_ESR`; `apic_eoi`; `atomic_inc(&irq_err_count)`; decode 8 error bits into the standard table ("Send CS error" ... "Illegal register address").

REQ-24: TSC-deadline microcode validation (`apic_validate_deadline_timer`):
- Match `boot_cpu_data` against `deadline_match[]` (Haswell-X, Broadwell-X/D, Skylake-X/L, Kaby Lake-L, ...) with required-microcode-revision driver_data.
- If `boot_cpu_data.microcode < required`: `setup_clear_cpu_cap(X86_FEATURE_TSC_DEADLINE_TIMER)`; pr_err FW_BUG with required version.
- `X86_FEATURE_XENPV`: clear feature unconditionally.
- `X86_FEATURE_HYPERVISOR` (non-Xen-PV): trust the hypervisor.

REQ-25: register_lapic_address(addr):
- `WARN_ON_ONCE(mp_lapic_addr)` (must only be set once).
- `mp_lapic_addr = addr`.
- If `!x2apic_mode`: `apic_set_fixmap(true)` (maps `FIX_APIC_BASE`, sets `apic_mmio_base = APIC_BASE`, reads BSP APIC ID).

REQ-26: init_apic_mappings:
- `apic_validate_deadline_timer()` (REQ-24) → "TSC deadline timer available" pr_info if true.
- If `x2apic_mode`: return (no MMIO needed).
- If `!smp_found_config`: `detect_init_APIC()` (REQ-27); if false: `apic_disable()` (install `apic_noop`).

REQ-27: detect_init_APIC:
- x86_64: if `!X86_FEATURE_APIC`: false; else `register_lapic_address(APIC_DEFAULT_PHYS_BASE = 0xFEE00000)`; true.
- x86_32: vendor-gated (AMD ≥ family 6 model 1, Hygon, Intel ≥ Pentium-Pro / family 5 with APIC); if `!X86_FEATURE_APIC` and `force_enable_local_apic`: `apic_force_enable(APIC_DEFAULT_PHYS_BASE)`.

REQ-28: PM suspend/resume:
- `lapic_suspend(data)`:
  - If `!apic_pm_state.active`: return 0.
  - Snapshot APIC_ID, APIC_TASKPRI, APIC_LDR, APIC_DFR, APIC_SPIV, APIC_LVTT, APIC_LVTPC (if maxlvt>=4), APIC_LVT0, APIC_LVT1, APIC_LVTERR, APIC_TMICT, APIC_TDCR, APIC_LVTTHMR (if >=5), APIC_LVTCMCI (if >=6) into `apic_pm_state`.
  - `mask_ioapic_entries()`; `disable_local_APIC()`; `irq_remapping_disable()`.
- `lapic_resume(data)`:
  - `mask_ioapic_entries()`; `legacy_pic->mask_all()`.
  - If `x2apic_mode`: `__x2apic_enable()`. Else: `__x2apic_disable()` if BIOS re-enabled; restore APICBASE base to `mp_lapic_addr`.
  - Write back snapshot in this order: LVTERR=ERROR_VEC|MASKED (first), ID, DFR, LDR, TASKPRI, SPIV, LVT0, LVT1, LVTTHMR (>=5), LVTCMCI (>=6), LVTPC (>=4), LVTT, TDCR, TMICT, ESR=0, read ESR, LVTERR=snapshot, ESR=0, read ESR.
  - `irq_remapping_reenable(x2apic_mode)`.

## Acceptance Criteria

- [ ] AC-1: `native_apic_msr_write(APIC_DFR | APIC_ID | APIC_LDR | APIC_LVR, v)` is a no-op (no MSR write).
- [ ] AC-2: `native_apic_msr_read(APIC_DFR)` returns `0xFFFFFFFF`.
- [ ] AC-3: For all non-blocked regs, `native_apic_msr_write/read` uses `APIC_BASE_MSR + (reg >> 4)`.
- [ ] AC-4: `setup_local_APIC` masks LVTERR **before** other LVTs when `maxlvt >= 3`.
- [ ] AC-5: `setup_local_APIC` sets TPR to 0x10 (vectors 0–31 ignored).
- [ ] AC-6: `setup_local_APIC` sets SPIV with `APIC_SPIV_APIC_ENABLED` and vector = `SPURIOUS_APIC_VECTOR` (0xff).
- [ ] AC-7: `apic_clear_isr` issues EOI for every ISR bit set on entry; warns on stale-IRR.
- [ ] AC-8: TSC-deadline mode: `lapic_next_deadline` writes `MSR_IA32_TSC_DEADLINE = rdtsc() + delta * 8`.
- [ ] AC-9: Oneshot non-deadline: `lapic_next_event` writes `APIC_TMICT = delta`.
- [ ] AC-10: `lapic_timer_shutdown` zeroes `MSR_IA32_TSC_DEADLINE` (deadline mode) or `APIC_TMICT` (non-deadline) **and** masks LVTT.
- [ ] AC-11: x2APIC enable: `MSR_IA32_APICBASE.X2APIC_ENABLE = 1`; disable: requires clearing both `X2APIC_ENABLE | XAPIC_ENABLE` then re-setting `XAPIC_ENABLE`.
- [ ] AC-12: `setup_APIC_eilvt` returns 0 only when cmpxchg succeeded and old/new are compatible per `eilvt_entry_is_changeable`.
- [ ] AC-13: BSP-only NMI (`apic_extnmi == APIC_EXTNMI_BSP`): only `cpu == 0` LVT1 unmasked.
- [ ] AC-14: `apic_intr_mode_select` returns `APIC_PIC` whenever `apic_is_disabled`.
- [ ] AC-15: `lapic_suspend` → `lapic_resume` restores all 14 snapshot fields and re-enables x2APIC if `x2apic_mode`.

## Architecture

```
struct LocalApic {                    // owns per-CPU state
  mmio_base: usize,                   // 0 in x2APIC mode
  x2apic_mode: bool,
  x2apic_state: X2ApicState,          // OFF | DISABLED | ON | ON_LOCKED
  intr_mode: IntrMode,                // PIC | VIRTUAL_WIRE | VIRTUAL_WIRE_NO_CONFIG | SYMMETRIC_IO | SYMMETRIC_IO_NO_ROUTING
  is_disabled: bool,
  extnmi: ExtNmi,                     // BSP | ALL | NONE
  boot_cpu_physical_apicid: u32,
  boot_cpu_apic_version: u8,
  timer_period: u32,                  // calibration result, ticks per HZ
  eilvt_offsets: [AtomicU32; APIC_EILVT_NR_MAX = 4],
  pm_state: ApicPmState,
}

static APIC: LocalApic;
per_cpu! { static EVENTS: ClockEventDevice; }
```

`Apic::msr_write(reg: u32, v: u32)`:
1. if `reg ∈ {APIC_DFR, APIC_ID, APIC_LDR, APIC_LVR}`: return.
2. `wrmsrq(APIC_BASE_MSR + (reg >> 4), v as u64)`.

`Apic::msr_read(reg: u32) -> u32`:
1. if `reg == APIC_DFR`: return `0xFFFFFFFF`.
2. `rdmsrq(APIC_BASE_MSR + (reg >> 4), &mut msr)`; return `msr as u32`.

`Apic::setup_local()`:
1. if `is_disabled`: `IoApic::disable_support()`; return.
2. `apic.driver.setup()`.
3. SPIV: clear `APIC_SPIV_APIC_ENABLED`; write back.
4. (x86_32) clear ESR 4x if disable_esr-driver and integrated.
5. `apic.driver.init_apic_ldr()` (logical-destination init).
6. TPR ← `(read & ~APIC_TPRI_MASK) | 0x10`.
7. `Apic::clear_isr()` (REQ-13).
8. SPIV ← `(read & ~APIC_VECTOR_MASK) | APIC_SPIV_APIC_ENABLED | SPURIOUS_APIC_VECTOR`.
9. `perf_events_lapic_init()`.
10. Configure LVT0 (REQ-6 step 10).
11. Configure LVT1 (REQ-6 step 11, with extnmi policy).
12. BSP only: `cmci_recheck()` if MCE-Intel built.

`Apic::clear_local()`:
1. Return if `!apic_accessible()`.
2. `maxlvt = get_maxlvt()`.
3. If `maxlvt >= 3`: LVTERR = `ERROR_APIC_VECTOR | APIC_LVT_MASKED`.
4. Mask LVTT, LVT0, LVT1 (and conditionally LVTPC, LVTTHMR, LVTCMCI).
5. Clean: write `APIC_LVT_MASKED` to LVTT/LVT0/LVT1/LVTERR/LVTPC.
6. If integrated and `maxlvt > 3`: ESR ← 0; read ESR.

`Apic::setup_lvtt(clocks, oneshot, irqen)`:
1. lvtt = `LOCAL_TIMER_VECTOR` | (PERIODIC if !oneshot) | (TSCDEADLINE if oneshot and feature) | (I82489DX_BASE_DIVIDER if not integrated) | (MASKED if !irqen).
2. `apic_write(APIC_LVTT, lvtt)`.
3. if `lvtt & APIC_LVT_TIMER_TSCDEADLINE`: `mfence`; return.
4. TDCR ← `(read & ~(DIV_1|DIV_TMBASE)) | DIV_16`.
5. if !oneshot: TMICT ← `clocks / APIC_DIVISOR`.

`Apic::next_event(delta, _evt) -> i32`:
1. `apic_write(APIC_TMICT, delta)`; return 0.

`Apic::next_deadline(delta, _evt) -> i32`:
1. `tsc = rdtsc()`.
2. `wrmsr(MSR_IA32_TSC_DEADLINE, tsc + (delta as u64) * TSC_DIVISOR)`.
3. return 0.

`Apic::timer_shutdown(evt) -> i32`:
1. if `evt.features & CLOCK_EVT_FEAT_DUMMY`: return 0.
2. v = `apic_read(APIC_LVTT) | APIC_LVT_MASKED | LOCAL_TIMER_VECTOR`.
3. `apic_write(APIC_LVTT, v)`.
4. if `v & APIC_LVT_TIMER_TSCDEADLINE`: `wrmsr(MSR_IA32_TSC_DEADLINE, 0)`.
5. else: `apic_write(APIC_TMICT, 0)`.
6. return 0.

`Apic::setup_eilvt(offset, vector, msg_type, mask) -> i32`:
1. `new = (mask << 16) | (msg_type << 8) | vector`.
2. `reg = APIC_EILVTn(offset)`.
3. `old = apic_read(reg)`.
4. `reserved = reserve_eilvt_offset(offset, new)`:
   - Loop: rsvd = atomic_load; vector_cur = rsvd & ~MASKED; if vector_cur && !changeable(vector_cur, new): return rsvd. atomic_cmpxchg(&offsets[offset], rsvd, new) → if changed: break.
5. if `reserved != new`: pr_err FW_BUG; return -EINVAL.
6. if `!changeable(old, new)`: pr_err FW_BUG; return -EBUSY.
7. `apic_write(reg, new)`; return 0.

`Apic::check_x2apic()`:
1. if `x2apic_enabled()`:
   - `x2apic_mode = 1`.
   - `x2apic_state = if x2apic_hw_locked() then ON_LOCKED else ON`.
   - `read_boot_cpu_id(true)`.
2. else if `!X86_FEATURE_X2APIC`: `x2apic_state = DISABLED`.

`Apic::x2apic_enable_msr()`:
1. read `MSR_IA32_APICBASE`; if `X2APIC_ENABLE` set: return.
2. write `MSR_IA32_APICBASE | X2APIC_ENABLE`.

`Apic::x2apic_disable_msr()`:
1. if `!X86_FEATURE_APIC`: return.
2. read `MSR_IA32_APICBASE`; if `!X2APIC_ENABLE`: return.
3. write `(msr & ~(X2APIC_ENABLE | XAPIC_ENABLE))` (disable both first).
4. write `(msr & ~X2APIC_ENABLE)` (re-enable xAPIC).

`Apic::intr_mode_select() -> IntrMode`:
1. if `is_disabled`: return `PIC`.
2. (x86_64) if `!X86_FEATURE_APIC`: `is_disabled = true`; return `PIC`.
3. (x86_32) if neither integrated nor SMP-found and not 82489DX: `is_disabled = true`; return `PIC`.
4. if `!smp_found_config`:
   - `IoApic::disable_support()`.
   - if `!acpi_lapic`: return `VIRTUAL_WIRE_NO_CONFIG`.
   - return `VIRTUAL_WIRE`.
5. if `!setup_max_cpus`: return `SYMMETRIC_IO_NO_ROUTING`.
6. return `SYMMETRIC_IO`.

`Apic::bsp_setup(upmode: bool)`:
1. `connect_bsp_APIC()`.
2. if upmode: `bsp_up_setup()` (= `reset_phys_cpu_present_map(boot_cpu_physical_apicid)`).
3. `setup_local()`.
4. `enable_IO_APIC()`.
5. `end_local_setup()` (= `setup_esr()` + `apic_pm_activate()`).
6. `irq_remap_enable_fault_handling()`.
7. `setup_IO_APIC()`.
8. `lapic_update_legacy_vectors()`.

`Apic::ap_setup()`:
1. `setup_local()`.
2. `end_local_setup()`.

`Apic::timer_idtentry(regs)`:
1. `old = set_irq_regs(regs)`.
2. `apic_eoi()`.
3. trace_entry.
4. `evt = this_cpu_ptr(&EVENTS)`.
5. if `evt.event_handler.is_none()`: pr_warn + shutdown + return.
6. `inc_irq_stat(apic_timer_irqs)`.
7. `evt.event_handler(evt)`.
8. trace_exit.
9. `set_irq_regs(old)`.

`Apic::suspend()`:
1. if `!pm_state.active`: return 0.
2. Snapshot 14 registers (REQ-28).
3. `mask_ioapic_entries()`; `disable_local()`; `irq_remapping_disable()`.

`Apic::resume()`:
1. if `!pm_state.active`: return.
2. `mask_ioapic_entries()`; `legacy_pic.mask_all()`.
3. if `x2apic_mode`: `x2apic_enable_msr()`. Else: disable x2APIC if BIOS re-enabled; restore APICBASE base.
4. Restore 14 registers in REQ-28 order.
5. `irq_remapping_reenable(x2apic_mode)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `msr_write_blocked_regs` | INVARIANT | per-msr_write: `APIC_DFR`, `APIC_ID`, `APIC_LDR`, `APIC_LVR` are no-op. |
| `msr_read_dfr_returns_minus_one` | INVARIANT | per-msr_read: `APIC_DFR` → `0xFFFFFFFF`. |
| `msr_index_in_range` | INVARIANT | per-msr_{read,write}: `APIC_BASE_MSR + (reg >> 4) ∈ [0x800, 0x83F]`. |
| `setup_local_mask_lvterr_first` | INVARIANT | per-clear_local: LVTERR masked before LVTT/LVT0/LVT1 when `maxlvt >= 3`. |
| `tpr_eq_0x10` | INVARIANT | per-setup_local: TPR == 0x10 on return. |
| `spiv_enabled_after_setup` | INVARIANT | per-setup_local: `APIC_SPIV_APIC_ENABLED` set on return. |
| `clear_isr_eoi_per_bit` | INVARIANT | per-check_and_eoi_isr: number of EOIs = popcount(ISR). |
| `tsc_deadline_writes_msr` | INVARIANT | per-next_deadline: writes `MSR_IA32_TSC_DEADLINE` exactly once. |
| `timer_shutdown_zeros_target` | INVARIANT | per-timer_shutdown: deadline mode zeroes MSR; else zeroes TMICT. |
| `eilvt_offset_bound` | INVARIANT | per-setup_eilvt: `offset < APIC_EILVT_NR_MAX = 4`. |
| `eilvt_atomic_reservation` | INVARIANT | per-reserve_eilvt_offset: cmpxchg loop terminates with consistent vector. |
| `x2apic_enable_idempotent` | INVARIANT | per-__x2apic_enable: already-enabled = no-op. |
| `x2apic_disable_two_phase` | INVARIANT | per-__x2apic_disable: clears both X2APIC|XAPIC then re-sets XAPIC. |
| `pm_suspend_resume_field_count` | INVARIANT | per-suspend+resume: 14 fields snapshotted, 14 restored. |
| `intr_mode_pic_when_disabled` | INVARIANT | per-intr_mode_select: `is_disabled` ⟹ `PIC`. |
| `nmi_bsp_only_when_extnmi_bsp` | INVARIANT | per-setup_local: LVT1 unmasked iff (`cpu==0 && extnmi != NONE`) ∨ `extnmi == ALL`. |

### Layer 2: TLA+

`arch/x86/apic.tla`:
- States: `OFF`, `XAPIC_ON`, `X2APIC_ON`, `X2APIC_ON_LOCKED`, `SOFT_DISABLED`, `HW_DISABLED`.
- Actions: `BiosBoot`, `CheckX2apic`, `TryEnableX2apic`, `SetupLocal`, `SoftDisable`, `Disable`, `Suspend`, `Resume`, `ApSetup`, `ApDisable`.
- Properties:
  - `safety_x2apic_no_downgrade_when_locked` — per-x2apic: `ON_LOCKED` cannot transition to `OFF`/`DISABLED`.
  - `safety_setup_clears_isr_first` — per-setup_local: `clear_isr` strictly before SPIV-enable.
  - `safety_lvterr_masked_first` — per-clear_local: LVTERR mask precedes other LVT masks.
  - `safety_tpr_accepts_only_vec32_plus` — per-setup_local: TPR = 0x10 always.
  - `safety_timer_deadline_xor_tmict` — per-next_deadline / next_event: writes deadline MSR XOR TMICT, never both.
  - `safety_eilvt_offset_uniqueness` — per-eilvt: each (offset, vector) tuple unique system-wide.
  - `safety_extnmi_bsp_unicast` — per-extnmi_bsp: only `cpu==0` has LVT1 unmasked NMI.
  - `liveness_calibration_terminates` — per-calibrate_APIC_clock: returns within `LAPIC_CAL_LOOPS = HZ/10` ticks or PIT/PMTMR fallback.
  - `liveness_resume_restores_complete_state` — per-resume: pm_state.active ⟹ all 14 fields written.
  - `liveness_isr_drain_finite` — per-clear_isr: at most `APIC_IR_BITS = 256` EOIs.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Apic::msr_read(APIC_DFR)` post: returns `0xFFFFFFFF` | `Apic::msr_read` |
| `Apic::msr_write` post: blocked-regs leave APICBASE-MSR-space unchanged | `Apic::msr_write` |
| `Apic::setup_local` post: SPIV-enabled, TPR=0x10, LVT0/LVT1 programmed per cpu+pic_mode+extnmi | `Apic::setup_local` |
| `Apic::clear_local` post: all LVT entries masked, ESR cleared if integrated | `Apic::clear_local` |
| `Apic::next_deadline` post: `MSR_IA32_TSC_DEADLINE = old_tsc + delta * 8` | `Apic::next_deadline` |
| `Apic::next_event` post: `APIC_TMICT = delta` | `Apic::next_event` |
| `Apic::timer_shutdown` post: LVTT masked, deadline-MSR-or-TMICT zeroed | `Apic::timer_shutdown` |
| `Apic::setup_eilvt` post: offset < 4, EILVTn(offset) = new on success | `Apic::setup_eilvt` |
| `Apic::x2apic_enable_msr` post: APICBASE.X2APIC_ENABLE = 1 | `Apic::x2apic_enable_msr` |
| `Apic::x2apic_disable_msr` post: APICBASE.X2APIC_ENABLE = 0 ∧ XAPIC_ENABLE = 1 | `Apic::x2apic_disable_msr` |
| `Apic::intr_mode_select` post: returns one of 5 enum values; deterministic given inputs | `Apic::intr_mode_select` |
| `Apic::suspend` + `Apic::resume` round-trip post: register state byte-identical (modulo errata-clears) | `Apic::{suspend, resume}` |
| `Apic::handle_spurious` post: EOI iff ISR-bit set; `irq_spurious_count` bumped | `Apic::handle_spurious` |
| `Apic::error_idtentry` post: ESR cleared, EOI'd, `irq_err_count` bumped | `Apic::error_idtentry` |

### Layer 4: Verus/Creusot functional

`Per-LAPIC programming: xAPIC-MMIO ↔ x2APIC-MSR register access equivalence; per-timer modes (oneshot via APIC_TMICT, periodic via APIC_LVT_TIMER_PERIODIC, TSC-deadline via MSR_IA32_TSC_DEADLINE) byte-identical to upstream; per-suspend/resume snapshot+restore round-trip functional` per-Intel SDM Vol. 3A Ch. 11 (Advanced Programmable Interrupt Controller), Ch. 10.5.4 (TSC-Deadline Mode), AMD APM Vol. 2 § 16.4 (Extended LVT), ACPI 6.5 § 5.2.12 (MADT).

## Hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

LAPIC reinforcement:

- **Per-msr_write blocks DFR/ID/LDR/LVR in x2APIC** — defense against per-#GP on read-only-MSR write.
- **Per-LVTERR-masked-first invariant** — defense against per-clear_local-induced APIC-error storm.
- **Per-ISR drain on setup_local** — defense against per-kexec stale-interrupt vector lockup.
- **Per-TPR=0x10 (block 0–31)** — defense against per-CPU-exception vector confusion via APIC.
- **Per-x2APIC HW-lock detection** — defense against per-`X2APIC_DISABLE` #GP storm post-resume.
- **Per-x2APIC-disable two-phase clear** — defense against per-spec-violation #GP.
- **Per-EILVT cmpxchg-reservation** — defense against per-AMD-MCE/IBS slot collision.
- **Per-NMI extnmi-bsp default** — defense against per-AP-NMI storm during early boot.
- **Per-`SETUP_RNG_SEED`-style explicit clear of pm_state on hotunplug** — defense against per-suspended-CPU register leak.
- **Per-`apic_validate_deadline_timer` microcode-rev gate** — defense against per-TSC-deadline errata data-corruption (Haswell/Broadwell/Skylake/Kaby Lake).
- **Per-`apic_check_and_eoi_isr` bounded by `APIC_IR_BITS=256`** — defense against per-stale-ISR DoS.
- **Per-spurious-handler ISR-verify before EOI** — defense against per-spurious-EOI of pending real interrupts.
- **Per-`acpi_mps_check`-gate at setup_arch** — defense against per-broken-firmware APIC misroute.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_KERNEXEC** — W^X enforcement for executable kernel mappings (APIC MMIO/MSR ops are RO+NX).
- **PAX_USERCOPY** — bounded copy_to/from_user when MSR snapshots or IRQ-stat counters surface to userspace.
- **PAX_REFCOUNT** — saturating refcount on per-CPU `lapic_events` clockevent and `apic_pm_state` references.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `apic_pm_state` snapshots on hotunplug to prevent register-leak.
- **PAX_UDEREF** — strict user-pointer access via SMAP/SMEP across LAPIC handlers.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `lapic_timer` ops, `apic.driver` hooks, and clockevent vtables.
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding for `mp_lapic_addr` / `apic_mmio_base` in /proc and dmesg.
- **GRKERNSEC_DMESG** — restrict APIC-error / spurious-vector / FW_BUG syslog output to CAP_SYSLOG.
- **MSR-write capability gate** — APIC MSR writes (`MSR_IA32_APICBASE`, `MSR_IA32_TSC_DEADLINE`, `MSR_IA32_XAPIC_DISABLE_STATUS`) gated by CAP_SYS_RAWIO surface.
- **PAX_LATENT_ENTROPY** — every interrupt vector stamps latent-entropy pool on IRQ entry path through the APIC.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization on the LAPIC timer-interrupt path that returns through `set_irq_regs`.
- **GRKERNSEC_KMEM** — block userspace `/dev/mem` mappings overlapping the LAPIC fixmap (`FIX_APIC_BASE`).
- **EILVT cmpxchg-reservation strict-bounds** — `offset < APIC_EILVT_NR_MAX = 4` enforced under PAX_USERCOPY-style index validation.
- **GRKERNSEC_PROC_ADD** — `/proc/interrupts` and per-CPU IRQ stats restricted to CAP_SYS_ADMIN.

Per-doc rationale: the LAPIC is the per-CPU interrupt + IPI + timer gatekeeper, so grsec/PaX hardening here protects every other subsystem's interrupt entry from being subverted via MSR write, stale-ISR leak across kexec, EILVT slot collision, or fixmap aliasing by userspace.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `arch/x86/kernel/apic/io_apic.c` IOAPIC (covered separately if expanded; cross-ref `kernel-platform.md` § APIC)
- `arch/x86/kernel/apic/ipi.c` IPI send paths (covered in `kernel-platform.md` Tier-3 § IPI)
- `arch/x86/kernel/apic/vector.c` vector allocation (covered in `kernel-platform.md` Tier-3 § Vector)
- `arch/x86/kernel/apic/msi.c` MSI/MSI-X composition (cross-ref via `__irq_msi_compose_msg` symbol-level here only)
- `arch/x86/kernel/apic/x2apic_*.c` x2APIC drivers (phys/cluster/uv) — symbol-level only
- `arch/x86/kernel/apic/apic_flat_64.c` / `apic_physflat.c` / `apic_noop.c` driver back-ends — symbol-level only
- `arch/x86/kernel/smpboot.c` AP-bringup orchestration (covered in `boot.md` Tier-3 § AP bringup)
- `arch/x86/kernel/nmi.c` NMI dispatch (covered in `kernel-platform.md` Tier-3 § NMI)
- `drivers/acpi/boot.c` MADT parsing (covered in `kernel-platform.md` Tier-3 § ACPI)
- `arch/x86/kernel/tsc.c` TSC calibration (covered in `kernel-platform.md` Tier-3 § Time)
- Implementation code
