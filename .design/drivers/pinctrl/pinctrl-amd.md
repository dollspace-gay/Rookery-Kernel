# Tier-3: drivers/pinctrl/pinctrl-amd.c — AMD AGPIO / KERNCZ pinctrl + GPIO driver

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pinctrl/00-overview.md
upstream-paths:
  - drivers/pinctrl/pinctrl-amd.c
  - drivers/pinctrl/pinctrl-amd.h
-->

## Summary

`pinctrl-amd.c` is the unified pinctrl + GPIO + IRQ-controller driver for AMD client SoCs — Carrizo, Stoney, Bristol Ridge, Raven, Renoir, Rembrandt, Phoenix, Hawk Point, Strix, and the embedded R-series — exposed as ACPI device `AMDI0030` / `AMDI0031`. The same hardware block (KERNCZ) provides ~184 GPIOs (4 banks: 63 + 64 + 56 + 32 = 215 pin numbers including reserved holes; ACPI passes a single MMIO window). Each pin has a per-pin 32-bit control register (offset = pin * 4) that combines pinmux (IOMUX function 0..3), pin configuration (bias pull-up / pull-down, drive strength, schmitt-trigger), GPIO direction + value, and per-pin interrupt control (edge/level, polarity, debounce, S0i3/S3/S4 wake source). A global "WAKE_INT_MASTER_REG" (0xFC) plus per-bank "WAKE_INT_STATUS_REGn" (0x2F8/0x2FC) latch wake events.

This Tier-3 covers ~1300 LOC: GPIO chip ops (direction, value, debounce, dbg_show), irq_chip (enable, disable, mask, unmask, set_type, set_wake, eoi, ack, the per-bank irq handler), pinctrl ops (groups via `pinctrl-generic`), pinmux ops (set_mux via IOMUX register), pinconf ops (pin_config_get/set on the per-pin register), s2idle ops via `acpi_s2idle_dev_ops` for AMD SOC s2idle integration, and the suspend/resume save/restore of per-pin registers.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct amd_gpio` | per-controller state: MMIO base, iomux_base, gpio_chip, pinctrl_dev, groups, irq, saved_regs | `drivers::pinctrl_amd::Controller` |
| `kerncz_pins[]` (~184 entries) | `pinctrl_pin_desc` array for all GPIOs | `Controller::PINS` |
| `kerncz_groups[]` | generated per-pin × per-function pingroups (4 mux functions per pin) | `Controller::GROUPS` |
| `amd_pinctrl_ops` | get_groups_count / _name / _pins via `pinctrl-generic` | `Controller::pctlops` |
| `amd_pmxops` | get_functions_count / _name / _groups / set_mux via IOMUX MMIO writes | `Controller::pmxops` |
| `amd_pinconf_ops` | pin_config_get / _set / _group_get / _group_set on per-pin reg | `Controller::confops` |
| `amd_gpio_irqchip` | `irq_chip` with enable / disable / mask / unmask / set_type / set_wake / eoi | `Controller::irqchip` |
| `amd_gpio_irq_handler` / `do_amd_gpio_irq_handler` | per-bank IRQ handler walks WAKE_INT_STATUS bits, dispatches to generic_handle_domain_irq | `Controller::irq_handler` |
| `amd_gpio_get_direction` / `_direction_input` / `_direction_output` | gpio_chip direction ops via OUTPUT_ENABLE_OFF bit | `Controller::direction_*` |
| `amd_gpio_get_value` / `_set_value` | gpio_chip value ops via PIN_STS_OFF / OUTPUT_VALUE_OFF bits | `Controller::get_value` / `set_value` |
| `amd_gpio_set_debounce` | DB_TMR_OUT_OFF / DB_TMR_OUT_UNIT_OFF / DB_CNTRL_OFF / DB_TMR_LARGE_OFF fields | `Controller::set_debounce` |
| `amd_gpio_set_config` | route gpio_chip set_config into pinconf_set (debounce, pull, drive) | `Controller::gpio_set_config` |
| `amd_gpio_irq_init` | mask all pins on probe; clear stale wake status | `Controller::irq_init` |
| `amd_gpio_check_pending` | s2idle helper — log pending wake before suspend completes | `Controller::check_pending` |
| `pinctrl_amd_s2idle_dev_ops` | `acpi_s2idle_dev_ops.prepare = amd_gpio_check_pending` hook | `Controller::s2idle_ops` |
| `amd_gpio_suspend` / `_hibernate` / `_resume` | save_regs[] backup + restore selected pins per `amd_gpio_should_save` | `Controller::suspend` / `_resume` |
| `amd_gpio_probe` / `_remove` | ACPI/platform attach: ioremap, irq_alloc_chip, pinctrl_register, gpiochip_add, devm_request_irq | `Controller::probe` / `_remove` |
| `amd_gpio_acpi_match[]` | `AMDI0030` + `AMDI0031` ACPI IDs | `Controller::ACPI_IDS` |

## Compatibility contract

REQ-1: Pinmux register: per-pin 32-bit control register at `base + pin * 4` with the bit layout encoded in `pinctrl-amd.h` (DB_TMR_OUT_OFF=0..3, DB_CNTRL_OFF=5..6, LEVEL_TRIG_OFF=8, ACTIVE_LEVEL_OFF=9..10, INTERRUPT_ENABLE_OFF=11, INTERRUPT_MASK_OFF=12, WAKE_CNTRL_OFF_S0I3=13, _S3=14, _S4=15, PIN_STS_OFF=16, DRV_STRENGTH_SEL_OFF=17..18, PULL_UP_ENABLE_OFF=20, PULL_DOWN_ENABLE_OFF=21, OUTPUT_VALUE_OFF=22, OUTPUT_ENABLE_OFF=23, SW_CNTRL_IN_OFF=24, SW_CNTRL_EN_OFF=25, WAKECNTRL_Z_OFF=27, INTERRUPT_STS_OFF=28, WAKE_STS_OFF=29).

REQ-2: IOMUX register window (separate from main pinctrl window, optional via `amd_get_iomux_res`): per-pin 8-bit register at `iomux_base + pin` containing the FUNCTION_MASK = bits [1:0]; FUNCTION_INVALID = 0xFF signals pin without iomux.

REQ-3: WAKE_INT_MASTER_REG (0xFC) bit 15 = INTERNAL_GPIO0_DEBOUNCE, bit 29 = EOI_MASK; written on probe to enable debounce timer and on every irq_eoi.

REQ-4: WAKE_INT_STATUS_REG0/1 (0x2F8/0x2FC) hold per-bank "pin has fired" bits; the IRQ handler reads them, then walks per-pin INTERRUPT_STS_OFF to find which pins actually need dispatch.

REQ-5: Set-mux: `amd_set_mux(pctldev, func, group)` — read IOMUX register, mask out `FUNCTION_MASK`, OR in the function selector encoded in `kerncz_groups`'s name; refuse if `FUNCTION_INVALID == 0xFF`.

REQ-6: Set-debounce: `amd_gpio_set_debounce` converts microseconds to DB_TMR_OUT field via four units (61us / 244us / 15.6ms / 62.4ms) selected via DB_TMR_OUT_UNIT_OFF + DB_TMR_LARGE_OFF, with DB_CNTRL_OFF picking glitch-preserve mode; refuse out-of-range debounces.

REQ-7: IRQ type: `amd_gpio_irq_set_type` writes LEVEL_TRIG_OFF + ACTIVE_LEVEL_OFF (3 modes: high, low, both edges); refuses unsupported `IRQ_TYPE_*` flags.

REQ-8: IRQ wake: `amd_gpio_irq_set_wake` toggles WAKE_CNTRL_OFF_S0I3 | WAKE_CNTRL_OFF_S3 (suspend) and WAKE_CNTRL_OFF_S4 (hibernate); used by ACPI s2idle.

REQ-9: Suspend/resume: `saved_regs[]` (one u32 per pin) backed on suspend for pins where `amd_gpio_should_save` returns true (interrupt-enabled, wake-enabled, or non-default mux), restored on resume in identical order.

REQ-10: s2idle integration: `pinctrl_amd_s2idle_dev_ops` registers with `acpi/x86/s2idle.c` so the AMD SOC s2idle flow checks for pending wake events before completing s2idle entry (avoids missed wake bug on Renoir).

REQ-11: Probe sequence: get ACPI/platform resource → devm_ioremap → optional iomux ioremap → irq_alloc_chip → `gpiochip_add_data` → `pinctrl_register_and_init` → `pinctrl_enable` → `devm_request_irq(IRQF_SHARED)`.

## Acceptance Criteria

- [ ] AC-1: Boot on a KERNCZ-class system (e.g. Lenovo ThinkPad Z13 / X13s-AMD): `dmesg | grep -i amd_gpio` shows probe success on `AMDI0030`.
- [ ] AC-2: `gpioinfo` lists ~184 pins under `gpiochip0` named `GPIO_0`..`GPIO_183` (with documented holes).
- [ ] AC-3: ACPI `\_GPE` event on a wake-enabled GPIO triggers `amd_gpio_irq_handler` and dispatches to the registered consumer ISR.
- [ ] AC-4: S3 suspend/resume cycle leaves all interrupt-enabled pins restored to their pre-suspend state (verified via `<debugfs>/gpio` snapshot).
- [ ] AC-5: s2idle entry on Renoir/Rembrandt with pending unhandled wake event aborts s2idle and logs the offending pin (s2idle_dev_ops integration).
- [ ] AC-6: `pinctrl-states` debugfs shows hogged `default` state for any board-defined pinctrl-amd consumers.

## Architecture

`Controller` lives in `drivers::pinctrl_amd`:

```
struct Controller {
  lock: RawSpinLock,
  base: NonNull<u8>,            // main per-pin control region (MMIO)
  iomux_base: Option<NonNull<u8>>, // optional separate iomux window
  groups: &'static [Group],
  ngroups: u32,
  pctrl: &Controller(pinctrl_dev),
  gc: GpioChip,
  hwbank_num: u32,              // 4 on KERNCZ
  res: Option<&Resource>,
  saved_regs: Option<KVec<u32>>,
  irq: u32,
  pdev: &PlatformDevice,
}
```

Probe `Controller::probe`:
1. `platform_get_resource(IORESOURCE_MEM)` → main MMIO region.
2. `devm_ioremap_resource(&pdev->dev, res)` → `base`.
3. `amd_get_iomux_res(gpio_dev)` — best-effort lookup of the optional IOMUX resource (separate ACPI region or compile-time offset).
4. `devm_kzalloc(saved_regs, npins * 4)`.
5. `gpiochip_add_data(&gpio_dev->gc, gpio_dev)`.
6. `pinctrl_register_and_init(&amd_pinctrl_desc, &pdev->dev, gpio_dev, &gpio_dev->pctrl)` + `pinctrl_enable`.
7. `gpiochip_add_pin_range(&gpio_dev->gc, dev_name, base=0, pin_base=0, npins)`.
8. `amd_gpio_irq_init` — mask all pins; clear all wake status bits.
9. `devm_request_irq(IRQF_SHARED | IRQF_NO_THREAD, amd_gpio_irq_handler, gpio_dev)`.
10. `amd_gpio_register_s2idle_ops()` (CONFIG_SUSPEND).

IRQ handler `do_amd_gpio_irq_handler(irq, gpio_dev)`:
1. `raw_spin_lock(&gpio_dev->lock)`.
2. For each bank (0..hwbank_num): read WAKE_INT_STATUS_REGn → if any bit set, walk every pin in bank and read per-pin reg for INTERRUPT_STS_OFF | WAKE_STS_OFF.
3. For each fired pin: `generic_handle_domain_irq(gc->irq.domain, pin)`.
4. After dispatch: write EOI_MASK in WAKE_INT_MASTER_REG to release the wake-master latch.
5. `raw_spin_unlock`.

Set-mux `amd_set_mux(pctldev, func, group)`:
1. Validate `func < num_functions`, `group < num_groups`.
2. If `iomux_base == NULL`: -ENOTSUPP.
3. Compute pin = `groups[group].pins[0]` (each group is one pin).
4. `readb(iomux_base + pin)`; if `(value == FUNCTION_INVALID)` return -EINVAL.
5. Mask out FUNCTION_MASK; OR in encoded function; `writeb` back.

Pinconf set `amd_pinconf_set(pctldev, pin, configs, num_configs)`:
1. For each config: decode PIN_CONFIG_BIAS_PULL_UP / _PULL_DOWN / _BIAS_DISABLE / _DRIVE_STRENGTH / _INPUT_DEBOUNCE.
2. Per-config: read per-pin reg, mask the relevant bits, OR in the new value, writel back.
3. Hold `raw_spin_lock` across each pin's RMW.

Suspend `amd_gpio_suspend`:
1. For each pin (0..npins): if `amd_gpio_should_save(gpio_dev, pin)`, `saved_regs[pin] = readl(base + pin*4)`.
2. `amd_gpio_check_pending()` walks WAKE_INT_STATUS for unhandled wake events and logs them (s2idle hint).

Resume `amd_gpio_resume`:
1. For each saved pin: `writel(saved_regs[pin], base + pin*4)`.
2. Re-arm EOI_MASK if needed.

## Hardening

- `IRQF_NO_THREAD` and `IRQF_SHARED` together: the AMD SoC IRQ may be shared with other ACPI events; per-handler raw_spinlock is the only protection needed.
- All MMIO accesses bounded: pin index multiplied by 4 against `npins * 4` MMIO range (validated at probe).
- Debounce field decode rejects out-of-range microsecond values.
- IRQ type validation rejects unsupported combinations (e.g. both rising and falling on a level-only pin).
- Per-pin `amd_gpio_should_save` skips uninteresting pins on suspend to minimize restore window.
- s2idle hook is the only writer of `pinctrl_amd_s2idle_dev_ops`; gated under CONFIG_SUSPEND.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `amd_gpio` and `saved_regs[]`; debugfs reads of pin state length-bounded.
- **PAX_KERNEXEC** — `pinctrl-amd.c` text W^X; all ops vtables (`amd_pinctrl_ops`, `amd_pmxops`, `amd_pinconf_ops`, `amd_gpio_irqchip`) `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel stack offset across `amd_gpio_irq_handler`, `amd_set_mux`, `amd_pinconf_set`, `amd_gpio_suspend`/`_resume`.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-pin / per-group references; module refcount on `amd_pinctrl_desc.owner` saturating.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `amd_gpio`, `saved_regs[]`, and any per-pin string buffers; refuse to expose stale wake-status across hot-unplug cycles.
- **PAX_UDEREF** — SMAP/PAN enforced on every debugfs / sysfs entry into pinctrl-amd; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `amd_pinctrl_ops`, `amd_pmxops`, `amd_pinconf_ops`, `amd_gpio_irqchip`, and `pinctrl_amd_s2idle_dev_ops` vtables kCFI-typed with `__ro_after_init` storage.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `amd_gpio_irq_handler`, `amd_set_mux`, MMIO base pointer behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict pinctrl-amd probe / s2idle-pending / suspend-restore banners to CAP_SYSLOG so attackers cannot map AGPIO layout via dmesg.
- **WAKE_INT MMIO bounded** — every `readl/writel(base + pin*4)` validates `pin < npins`; WAKE_INT_STATUS / WAKE_INT_MASTER offsets are constants policed against the MMIO region size at probe.
- **Debounce-counter validation** — `amd_gpio_set_debounce` rejects out-of-range microsecond values and out-of-range DB_TMR_OUT_UNIT_OFF / DB_TMR_LARGE_OFF combinations; refuse to silently program nonsense timing.
- **Interrupt-routing RO after init** — IOMUX register writes only allowed via canonical `amd_set_mux` path; debugfs / sysfs writes to per-pin reg gated on CAP_SYS_ADMIN and never allowed to bypass strict-mode + iomux re-validation.
- **EOI_MASK bookkeeping** — EOI write at the end of the IRQ handler is unconditional and bounded; any double-EOI / missed-EOI sequence is logged + the handler bails to dispatch path.
- **s2idle hook auditable** — `pinctrl_amd_s2idle_dev_ops` registration logged; pending-wake check rate-limited and bounded.

Rationale: pinctrl-amd is the policy enforcement point for every AGPIO pin on AMD client SoCs — including pins carrying laptop lid, fingerprint reader interrupt, TPM IRQ, and embedded controller signaling. A corrupted set_mux, a missed debounce-bounds check, or an out-of-bounds MMIO write can route arbitrary internal signals to externally accessible pads or mask security-sensitive wake events. Refcount saturation, MMIO range policing, debounce validation, and kCFI on vtable dispatch turn the AMD pinctrl driver into a structural pad-routing boundary on every Ryzen client machine.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mmio_no_oob` | OOB | every `readl/writel(base + pin*4)` has `pin < npins`. |
| `iomux_no_oob` | OOB | every `readb/writeb(iomux_base + pin)` has `pin < iomux_npins`. |
| `saved_regs_no_oob` | OOB | suspend/resume index never exceeds `saved_regs.len()`. |
| `irq_dispatch_no_uaf` | UAF | per-pin `irq_data` lifetime tied to `gpiochip`; refuses dispatch after `gpiochip_remove`. |

### Layer 2: TLA+

`models/pinctrl_amd/irq_dispatch.tla`: proves that concurrent `irq_handler` and `irq_set_type` / `irq_set_wake` on the same pin observe consistent state under raw_spinlock — no torn-write of per-pin control register.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `amd_set_mux` post: IOMUX register reads back the programmed function value | `Controller::set_mux` |
| `amd_gpio_irq_handler` post: every fired pin's INTERRUPT_STS_OFF cleared OR handler logs missed-dispatch | `Controller::irq_handler` |
| `amd_gpio_resume` post: every saved pin's control register restored byte-identical | `Controller::resume` |

### Layer 4: Verus/Creusot functional

`probe → set_mux(group, func) → request_irq → suspend → resume → pin still has func mux + still has IRQ wired → remove` leaves the SoC with no dangling MMIO mappings and no lingering wake-status bits. Encoded as Verus invariant chained with `pinctrl/00-overview.md`.

## Out of Scope

- Intel pinctrl drivers (covered in future `pinctrl-intel.md`)
- AMD server (EPYC) GPIO via `pinctrl-amdisp.c` (future companion doc)
- ACPI s2idle constraint logic in `acpi/x86/s2idle.c` (future)
- 32-bit-only paths
- Implementation code
