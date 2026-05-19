# Tier-3: drivers/irqchip/irq-gic.c — ARM GIC v1/v2 (Distributor + per-CPU CPU Interface)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/irqchip/00-overview.md
upstream-paths:
  - drivers/irqchip/irq-gic.c
  - drivers/irqchip/irq-gic-common.c
  - drivers/irqchip/irq-gic-common.h
  - include/linux/irqchip/arm-gic.h
-->

## Summary

ARM Generic Interrupt Controller v1/v2 (GICv1, GICv2, plus GIC-400 and Cortex-A9 MPCore variants). The hardware splits into a single Distributor (GICD) that prioritises + routes interrupts to CPU interfaces, and one CPU Interface (GICC) per processor that arbitrates the currently-presented interrupt + acknowledges/EOIs. Interrupt sources are partitioned into SGI (Software-Generated Interrupts, hwirq 0–15, used for IPIs), PPI (Private Peripheral Interrupts, hwirq 16–31, per-CPU local), and SPI (Shared Peripheral Interrupts, hwirq 32–1019, system-wide).

This Tier-3 covers `irq-gic.c` (~1700 lines: probe, distributor + cpu-iface init, irq_chip ops, irq-domain ops, PM save/restore, BL-switcher mux, cascading GIC handling) and the small shared helpers in `irq-gic-common.c`.

Pre-dates GICv3 which is system-register-based and is the model for modern ARM64 SoCs (see `irq-gic-v3.md`). GICv1/v2 are MMIO-only and remain on legacy ARM SoCs (Cortex-A7/A9/A15, older Qualcomm MSM, older STMicro, ARM Versatile/Realview reference boards).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct gic_chip_data` | per-GIC-instance control block (dist + cpu bases, domain, gic_irqs count) | `drivers::irqchip::gic::Gic` |
| `gic_of_init(node, parent)` | DT init: map MMIO, init distributor + CPU iface, register domain | `Gic::of_init` |
| `__gic_init_bases(gic, handle)` | shared init for OF/ACPI: domain alloc, irq_chip setup, set_handle_irq for root | `Gic::init_bases` |
| `gic_handle_irq(regs)` | arch-entry root handler: read GICC_IAR, dispatch via `generic_handle_domain_irq`, write GICC_EOIR | `Gic::handle_irq` |
| `gic_mask_irq(d)` / `gic_unmask_irq(d)` | write GICD_ICENABLER / GICD_ISENABLER bit | `Gic::mask` / `Gic::unmask` |
| `gic_eoi_irq(d)` / `gic_eoimode1_eoi_irq(d)` | write GICC_EOIR (split-EOI for KVM-VGIC) | `Gic::eoi` / `Gic::eoi_mode1` |
| `gic_set_type(d, type)` | write GICD_ICFGR for level/edge selection | `Gic::set_type` |
| `gic_set_affinity(d, mask, force)` | write GICD_ITARGETSR byte for SPI routing | `Gic::set_affinity` |
| `gic_raise_softirq(mask, irq)` | write GICD_SGIR to fire SGI on target CPUs | `Gic::raise_softirq` |
| `gic_irq_domain_translate(d, fwspec, &hwirq, &type)` | decode 3-cell DT spec `<type hwirq flags>` | `Gic::translate` |
| `gic_irq_domain_map(d, virq, hwirq)` / `gic_irq_domain_alloc(d, virq, nr, arg)` | per-virq alloc, install `gic_chip` + flow handler | `Gic::domain_map` / `Gic::domain_alloc` |
| `gic_cpu_init(gic)` | per-CPU init: enable PPIs + SGIs, set priority mask, enable GICC | `Gic::cpu_init` |
| `gic_dist_init(gic)` | distributor init: configure SPIs as level + target CPU0, enable GICD | `Gic::dist_init` |
| `gic_cpu_save(gic) / gic_cpu_restore(gic) / gic_dist_save(gic) / gic_dist_restore(gic)` | CPU_PM-cycle save/restore | `Gic::cpu_save` / `cpu_restore` / `dist_save` / `dist_restore` |
| `gic_of_init_child(dev, gic, irq)` | cascaded GIC child registered under a parent's chained handler | `Gic::of_init_child` |
| `IRQCHIP_DECLARE(gic_400, "arm,gic-400", gic_of_init)` (+ 8 sibling decls) | per-compat-string registrations | `Gic::compat_table` |

## Compatibility contract

REQ-1: One Distributor (GICD) per system at the address declared by the first `reg = <...>` cell in the DT `interrupt-controller` node; up to `CONFIG_ARM_GIC_MAX_NR` cascaded instances supported on legacy multi-cluster SoCs.

REQ-2: One CPU Interface (GICC) per CPU, declared by the second `reg` cell; on banked-implementations the same address is aliased per-CPU, on non-banked ("frankengic") implementations each CPU has its own MMIO range and `percpu_offset` is set.

REQ-3: Three interrupt classes: SGI (0–15, IPIs, banked per-CPU), PPI (16–31, per-CPU timers/PMU, banked per-CPU), SPI (32 .. GICD_TYPER.ITLinesNumber * 32 − 1, system-wide peripherals).

REQ-4: DT cell shape — 3 cells: `<type hwirq flags>` where type=0 → SPI, type=1 → PPI, `hwirq` is offset within the class, `flags` is IRQ_TYPE_LEVEL_HIGH/LOW/EDGE_RISING/FALLING.

REQ-5: Per-SPI affinity programmed via GICD_ITARGETSRn — each byte selects up to 8 CPU targets via bitmap (GICv2: max 8 CPUs per GIC).

REQ-6: SGI dispatch via GICD_SGIR — target-list-filter (b00) + 8-bit CPU mask + SGI number, fires same SGI on multiple CPUs in one MMIO write.

REQ-7: Two acknowledge modes — EOImode0 (single write to GICC_EOIR both priority-drops + deactivates) and EOImode1 (priority-drop via GICC_EOIR, deactivate via GICC_DIR; used by KVM-VGIC for guest IRQ injection).

REQ-8: Priority mask programmed in GICC_PMR; only IRQs with priority numerically lower than PMR get delivered (lower=higher-priority in GIC arithmetic).

REQ-9: BL-switcher (big.LITTLE) mux support: when a big↔LITTLE migration happens, the SPI ITARGETSR for the outbound CPU is rewritten to the inbound CPU; protected by `cpu_map_lock`.

REQ-10: Non-banked ("frankengic") quirk: some SoCs (some Qualcomm MSMs, some Marvell Armada XP) implement GICC at distinct addresses per-CPU; driver activates a static-key (`frankengic_key`) and uses `percpu_base` for accesses.

REQ-11: PM (`CONFIG_CPU_PM` / `CONFIG_ARM_GIC_PM`): on CPU-cluster power-down, save SPI-enable/active/conf/target + PPI-enable/active/conf to per-instance buffers; restore on power-up.

REQ-12: ACPI MADT (`gic_acpi_init`) — invoked from `arch/arm64/kernel/acpi.c` for ACPI-boot ARM64 platforms that still have GICv2 (rare; mostly QEMU).

## Acceptance Criteria

- [ ] AC-1: Boot 32-bit ARM virt + GICv2 (`qemu-system-arm -machine virt`): dmesg shows distributor + CPU iface init, `cat /proc/interrupts` lists per-CPU timer (PPI 11/13) + virtio peripherals (SPIs).
- [ ] AC-2: SMP boot on 4-CPU Cortex-A15 reference platform: each CPU starts; per-CPU SGIs deliver `IPI_RESCHEDULE` correctly.
- [ ] AC-3: KVM-VGIC tests on host with EOImode1 path active — `tools/testing/selftests/kvm/aarch64/vgic_irq` passes with `KVM_DEV_ARM_VGIC_V2`.
- [ ] AC-4: CPU hotplug cycle — `echo 0 > /sys/devices/system/cpu/cpu1/online; echo 1 > ...` — per-CPU PPIs re-armed without leaks.
- [ ] AC-5: `/proc/irq/N/smp_affinity 0x04` reroutes an SPI to CPU2; observed in GICD_ITARGETSR readback.
- [ ] AC-6: CPU-PM cycle (suspend/resume) restores all SPI enable/conf/target bits; post-resume interrupts continue to deliver.
- [ ] AC-7: BL-switcher migration: SPI target rewritten when CPU2(big) hands off to CPU2(LITTLE); no stalled-IRQ regression.

## Architecture

`Gic` (Rust analogue):

```
struct Gic {
  dist_base: union { common: NonNull<u8>, per_cpu: PerCpu<NonNull<u8>> },
  cpu_base:  union { common: NonNull<u8>, per_cpu: PerCpu<NonNull<u8>> },
  raw_dist_base: NonNull<u8>,
  raw_cpu_base:  NonNull<u8>,
  percpu_offset: u32,
  domain: Arc<IrqDomain>,
  gic_irqs: u32,                 // total SPIs supported
  saved: Option<GicPmState>,     // suspend/resume buffers
}
```

Init flow `Gic::init_bases`:
1. Memremap `dist_base` and `cpu_base`; if `percpu_offset != 0` allocate per-CPU base array.
2. Read GICD_TYPER to compute `gic_irqs = (TYPER.ITLinesNumber + 1) * 32`, clamp to 1020 (spec max for GICv2).
3. `irq_domain_create_linear(fwnode, gic_irqs, &gic_irq_domain_hierarchy_ops, gic)` (or simple `_create_legacy` for legacy DT path).
4. `gic_dist_init(gic)`: write GICD_CTLR=0, mark all SPIs target=CPU0, level-trigger, priority 0xa0, then GICD_CTLR=1.
5. `gic_cpu_init(gic)`: enable PPIs 16–31, set GICC_PMR=0xf0, write GICC_CTLR enable + EOImodeNS bit.
6. If root: `set_handle_irq(gic_handle_irq)`; if cascaded: `irq_set_chained_handler(parent_irq, gic_handle_cascade_irq)`.
7. Register cpuhp callbacks for per-CPU SGI/PPI init/teardown on hotplug.

Hot-path IRQ dispatch `gic_handle_irq(regs)`:
1. `irqnr = readl(GICC_IAR) & GICC_IAR_INT_ID_MASK`.
2. If `irqnr < 15`: SGI — read `sgi_intid` for the per-CPU IPI vector; write GICC_EOIR(irqnr); dispatch IPI.
3. Else if `irqnr < 1020`: regular IRQ — `generic_handle_domain_irq(gic->domain, irqnr)` which calls the flow-handler → handler chain.
4. Else (spurious): just return.

`irq_chip` ops:
- `irq_mask` → `writel(BIT(hwirq%32), GICD_ICENABLER + (hwirq/32)*4)`
- `irq_unmask` → `writel(BIT(hwirq%32), GICD_ISENABLER + (hwirq/32)*4)`
- `irq_eoi` → `writel(hwirq, GICC_EOIR)` (or split via GICC_DIR in mode1)
- `irq_set_type` → read-modify-write GICD_ICFGR (2 bits per IRQ: level vs edge); requires disabling the IRQ first
- `irq_set_affinity` → byte-write to GICD_ITARGETSR; bitmap restricted to CPUs in `gic_cpu_map[]` mapping logical→GIC CPU IDs

Cascading: `gic_handle_cascade_irq(desc)` reads child GICC_IAR, calls `generic_handle_domain_irq(child->domain, ...)`, writes EOIR to child, then `chained_irq_exit(parent_chip, parent_desc)`.

PM save (`gic_cpu_save`/`gic_dist_save`): per-instance `gic_chip_data` holds `saved_spi_enable[DIV_ROUND_UP(1020,32)]` etc.; written on `CPU_PM_ENTER` notifier, restored on `CPU_PM_EXIT`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `gic_irqs_no_oob` | OOB | `hwirq < gic->gic_irqs` enforced before every GICD_ register byte/word offset computation. |
| `cpu_map_in_range` | OOB | per-CPU `gic_cpu_map[]` entries < NR_GIC_CPU_IF (=8) for GICv2. |
| `sgi_target_valid` | OOB | GICD_SGIR target-list mask AND'd with `cpu_possible_mask`. |
| `pm_buf_sized` | OOB | save/restore buffer sized to `DIV_ROUND_UP(1020, X)` always; per-CPU PPI buffers allocated via `alloc_percpu`. |

### Layer 2: TLA+

`models/irqchip/gic_eoi.tla` (future): proves single-EOI vs split-EOI orderings observe deactivate-after-handler invariant across KVM-VGIC injection paths.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Gic::set_type` post: `GICD_ICFGR.bits[2*(hwirq%16)+1]` matches requested edge/level | `Gic::set_type` |
| `Gic::set_affinity` post: GICD_ITARGETSR byte equals `cpumask_to_targetlist(mask)` | `Gic::set_affinity` |
| `Gic::handle_irq` post: every IAR read is followed by exactly one EOIR write (or DIR if mode1) | `Gic::handle_irq` |

### Layer 4: Verus/Creusot functional

Encoded: writing to GICD_ICENABLER for hwirq H makes a subsequent firing of H not delivered to any CPU until GICD_ISENABLER re-arms it; relates the masked/unmasked transition to the visible delivery.

## Hardening

- GICD_/GICC_ MMIO offsets bounded by `gic_irqs`; refuse out-of-range hwirq.
- `gic_cpu_map[8]` is `__read_mostly` and populated only at init; lookup never trusts external indices.
- `IRQCHIP_DECLARE` table in `.init.rodata`, discarded post-`free_initmem`.
- Per-CPU PM buffers allocated via `alloc_percpu`; size known at compile time.
- `cpu_map_lock` (raw spinlock) protects BL-switcher mux of GICD_ITARGETSR.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `gic_chip_data` slab + per-CPU PM buffers whitelisted; `/proc/interrupts` rows strictly bounded copies.
- **PAX_KERNEXEC** — `gic_handle_irq`, `gic_chip` ops, distributor/cpu init paths in W^X `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `gic_handle_irq` and chained-handler entries so per-IRQ stack-relative gadgets cannot be reused.
- **PAX_REFCOUNT** — saturating refs on per-GIC `irq_domain` and per-CPU cpuhp callbacks; overflow trap defeats double-remove races on hotplug.
- **PAX_MEMORY_SANITIZE** — zero-on-free for PM save buffers so retired SPI-enable/conf bitmaps cannot leak to a successor allocation.
- **PAX_UDEREF** — SMAP/PAN on `/proc/irq/*/smp_affinity` and sysfs writes that copy bitmasks from user.
- **PAX_RAP / kCFI** — `gic_chip` ops (`irq_mask`/`unmask`/`eoi`/`set_type`/`set_affinity`), domain ops (`map`/`alloc`/`translate`), and chained-handler trampolines marked kCFI-typed.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `gic_data[]`, `gic_cpu_map[]`, and per-instance MMIO base pointers behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict GIC init banners, distributor-warning printks, and BL-switcher migration logs to CAP_SYSLOG.
- **GICD_* MMIO bounded** — every GICD_/GICC_ register offset clamped against `gic_irqs` and against the memremap'd region length; refuse reads/writes past the mapped extent.
- **SGI/PPI/SPI numbering enforced** — translate rejects hwirq outside `[0, gic_irqs)`; SGI calls validate hwirq < 16; PPI alloc validates hwirq < 32; defense against DT-shape confusion.
- **Distributor lock IRQ-safe** — `cpu_map_lock` taken with `raw_spinlock_irqsave` along BL-switcher and SGI-raise paths; defense against nested-IRQ deadlock and lockdep-class violations.
- **GICD_SGIR target-list policed** — SGI target mask AND'd with `cpu_online_mask` before MMIO write; refuse to fire SGIs at offline CPUs.
- **EOImode1 KVM gate** — GICC split-EOI mode only enabled when `supports_deactivate_key` is set during init; cannot be flipped at runtime by guest action.
- **PM save buffer validated on restore** — checksum / generation-counter on `saved_spi_*` arrays; refuse restore if the buffer doesn't match the active GIC instance signature.

Rationale: GIC sits between every device IRQ and the CPU interrupt-handler — a corrupted GICD_ITARGETSR or stale EOI write reroutes physical interrupts to attacker-chosen CPUs or stalls the handler chain entirely. kCFI on chip-ops, MMIO clamping, SGI cpumask AND, and PM-buffer validation turn the legacy GIC driver into a structurally enforced delivery path.

## Open Questions

- (none at this Tier-3 level)

## Out of Scope

- GICv3 + ITS (covered in `irq-gic-v3.md`)
- GICv5 (settling upstream; future Tier-3)
- GIC-V2M MSI bridge (covered as part of `irq-gic-v3.md` since it shares plumbing)
- KVM-VGIC internals (covered in `virt/kvm/00-overview.md`)
- 32-bit-only PPC GIC ports (not used by Rookery)
- Implementation code
