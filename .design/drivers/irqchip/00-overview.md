# Tier-2: drivers/irqchip — interrupt controller drivers (irq-domain hierarchy, IRQCHIP_DECLARE, per-arch root + cascaded chips)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/irqchip/
  - include/linux/irqchip.h
  - include/linux/irqdomain.h
  - include/linux/irqdomain_defs.h
-->

## Summary

`drivers/irqchip/` is the kernel's collection of per-platform interrupt-controller drivers: ARM GIC v1/v2/v3/v4/v5, MIPS GIC, RISC-V PLIC/APLIC/IMSIC, x86 i8259/IO-APIC (kept here as a stub — the bulk lives under `arch/x86/`), Apple AIC, Broadcom L1/L2 cascades, BCM2835 (Raspberry Pi), the various per-SoC interrupt aggregators (Atmel AIC, Sunxi NMI, Renesas INTC, Loongson LIOINTC, etc.), and the GICv3 ITS (Interrupt Translation Service) for MSI/MSI-X-style message-signalled interrupts on ARM64.

The subsystem is the glue between the generic IRQ core (`kernel/irq/`, see `kernel/irq/00-overview.md`) and per-platform hardware. Each driver registers one or more `irq_domain` structures via `IRQCHIP_DECLARE(name, compat, init_fn)` (DT) or `IRQCHIP_ACPI_DECLARE(...)` (ACPI MADT), declares its `irq_chip` callbacks (`irq_mask`/`unmask`/`eoi`/`set_type`/`set_affinity`), and either acts as a root chip (sets `set_handle_irq()` for the asm entry path) or chains under a parent domain via `irq_set_chained_handler()`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `IRQCHIP_DECLARE(name, compat, fn)` | per-driver compat-string → init-fn registration in `__irqchip_of_table` | `drivers::irqchip::IrqchipTable` |
| `IRQCHIP_ACPI_DECLARE(name, subtable, validate, data, fn)` | ACPI MADT subtable-based registration | `drivers::irqchip::IrqchipAcpiTable` |
| `irq_domain_create_linear(fwnode, size, ops, host_data)` | linear hwirq → virq map | `IrqDomain::create_linear` |
| `irq_domain_create_tree(fwnode, ops, host_data)` | sparse radix-tree map for huge hwirq spaces (ITS, MBI) | `IrqDomain::create_tree` |
| `irq_domain_create_hierarchy(parent, flags, size, fwnode, ops, host_data)` | hierarchical domain (root → MSI → device) | `IrqDomain::create_hierarchy` |
| `irq_domain_alloc_irqs(domain, nr_irqs, node, arg)` | alloc N contiguous virqs from a domain | `IrqDomain::alloc_irqs` |
| `irq_create_mapping(domain, hwirq)` | per-hwirq → per-virq lazy alloc | `IrqDomain::create_mapping` |
| `irq_find_matching_fwspec(fwspec, bus_token)` | resolve fwspec to domain | `IrqDomain::find_matching_fwspec` |
| `struct irq_domain_ops` | `.match`/`.alloc`/`.free`/`.translate`/`.activate`/`.deactivate`/`.map`/`.xlate` vtable | `IrqDomainOps` trait |
| `struct irq_chip` | per-controller callbacks vtable | `IrqChip` trait |
| `set_handle_irq(fn)` | root chip registers asm-entry dispatch | `RootIrqChip::set_handler` |
| `irq_set_chained_handler_and_data(virq, fn, data)` | cascaded chip attaches to parent virq | `IrqChip::set_chained_handler` |
| `generic_handle_domain_irq(domain, hwirq)` | per-domain hwirq → flow handler | `IrqDomain::handle_domain_irq` |
| `handle_fasteoi_irq` / `handle_level_irq` / `handle_edge_irq` | per-flow handlers from `kernel/irq/chip.c` | (cross-ref `kernel/irq/00-overview.md`) |
| `platform_irqchip_probe(pdev)` | driver-model probe wrapper for late-loaded drivers | `Platform::irqchip_probe` |

## Compatibility contract

REQ-1: `IRQCHIP_DECLARE` entries land in the `__irqchip_of_table` linker section; `irqchip_init()` (called from `init/main.c::start_kernel()` via `of_irq_init`) walks DT for matching compatible strings and invokes the init-fn in dependency order (root chips first, cascaded chips after their parent is up).

REQ-2: Per-driver init-fn signature is `int (*)(struct device_node *node, struct device_node *parent)` for DT and `int (*)(union acpi_subtable_headers *header, const unsigned long end)` for ACPI.

REQ-3: Each driver registers at least one `irq_domain` with a unique `fwnode_handle` (DT node or ACPI subtable surrogate) so child DT nodes' `interrupts = <...>` properties resolve via `irq_find_matching_fwspec`.

REQ-4: `irq_domain_ops::translate` (or legacy `.xlate`) decodes per-DT-binding cell-count + cell-shape (e.g. GIC's 3-cell `<type hwirq flags>`) into `(hwirq, type)`.

REQ-5: `irq_domain_ops::alloc/.free` allocate virqs lazily on `irq_create_mapping` or hierarchically when MSI/MSI-X consumers need a contiguous range.

REQ-6: A root chip MUST install an arch-entry handler via `set_handle_irq()` (ARM/MIPS/RISC-V) or `handle_arch_irq` pointer (arch-specific) and that handler MUST call `generic_handle_domain_irq()` for each pending hwirq.

REQ-7: Cascaded chips MUST `irq_set_chained_handler_and_data(parent_virq, dispatch_fn, this_chip)` and the dispatch-fn MUST call `chained_irq_enter/exit` to bracket the cascade.

REQ-8: Per-CPU local interrupts (PPI on GIC, GIC_LOCAL_INT_* on MIPS GIC, IPI vectors) are typically allocated via `irq_create_mapping` to a per-CPU domain or a hierarchical IPI domain.

REQ-9: `irq_chip::flags` declares per-driver semantics: `IRQCHIP_SET_TYPE_MASKED`, `IRQCHIP_EOI_IF_HANDLED`, `IRQCHIP_MASK_ON_SUSPEND`, `IRQCHIP_SKIP_SET_WAKE`, etc.

REQ-10: Drivers compiled as builtin from `drivers/irqchip/Kconfig` — irqchip drivers are NEVER loadable modules because root chips are needed before `kernel_init` runs.

REQ-11: `acpi_probe_entry::probe_subtbl` for ACPI replaces DT for x86 + ARM64 ACPI boot.

REQ-12: `irq_set_default_domain(d)` may be set by the root chip so legacy non-DT consumers resolve via `irq_create_mapping(NULL, hwirq)`.

## Acceptance Criteria

- [ ] AC-1: Boot on ARM64 reference platform (Juno R2 or QEMU virt + GICv3) — `dmesg | grep -i 'GICv3\|ITS\|GIC '` shows distributor + N redistributors + ITS init.
- [ ] AC-2: Boot on 32-bit ARM virt + GICv2 — `dmesg | grep -i 'GIC: '` shows distributor + CPU interface init.
- [ ] AC-3: Boot on MIPS Malta or Boston board — `dmesg | grep -i 'mips-gic\|gic'` shows MIPS GIC init with reported shared + local interrupt counts.
- [ ] AC-4: `cat /proc/interrupts` lists per-CPU PPIs + per-shared SPIs + per-ITS-LPI MSIs without holes; affinity-write via `/proc/irq/N/smp_affinity` updates redistributor target.
- [ ] AC-5: PCIe NVMe MSI-X allocation succeeds on GICv3+ITS (verified by `cat /proc/interrupts | grep nvme`).
- [ ] AC-6: KVM IPI passthrough test on GICv4-capable hardware (or GICv4.1 + KVM_DEV_TYPE_ARM_VGIC_V4) exercises vSGI hot-path.
- [ ] AC-7: kselftest `tools/testing/selftests/kvm/aarch64/vgic_irq` exercises VGIC injection paths against irqchip driver.

## Architecture

Per-driver shape:

```
IRQCHIP_DECLARE(name, "vendor,part", init_fn)
   ↓ link-time section __irqchip_of_table
init_fn(node, parent):
   - of_iomap(node, ...) for MMIO
   - request_irq() / set_handle_irq() for root, or
     irq_set_chained_handler() for cascaded
   - irq_domain_create_{linear,tree,hierarchy}(...)
   - register per-CPU notifier (cpuhp_setup_state) for PPI / local irq init
```

Three structural roles:

1. **Root chip** — first chip on the IRQ delivery path; the arch asm-entry calls into its handler (`set_handle_irq(fn)`). Root chips own the canonical `irq_domain` mapped from DT root. Examples: GIC, GICv3, MIPS GIC, RISC-V PLIC, Apple AIC.

2. **Cascaded chip** — subordinate; its IRQ output line is wired into an input of a parent (root or another cascaded) chip. Linux models this by allocating a virq from the parent for the cascade-input and installing a chained handler that demuxes per-child hwirq + calls `generic_handle_domain_irq()`. Examples: BCM2836 local IRQs cascaded into BCM2835, OMAP-INTC cascaded into ARM legacy controller.

3. **Hierarchical domain (MSI/MSI-X)** — for message-signalled interrupts, drivers build a parent-child domain chain: `device_msi_domain -> platform_msi_parent_domain -> ITS/MBI -> GIC`. Each layer's `irq_chip` covers a slice of the configuration (device writes MSI-X table at the leaf, ITS programs hardware translation at the parent, GIC controls per-CPU delivery at root).

`struct irq_domain` (`include/linux/irqdomain.h`) holds:
- `ops` — `irq_domain_ops` vtable
- `hwirq_max` — top hwirq number
- `revmap` (linear or radix-tree) for hwirq→virq lookup
- `host_data` — per-chip pointer (`struct gic_chip_data` etc.)
- `parent` — parent domain in hierarchy
- `flags` — `IRQ_DOMAIN_FLAG_HIERARCHY`, `_IPI_PER_CPU`, `_MSI`, `_NONCORE`, etc.

`struct irq_domain_ops`:
```
{ match, select, alloc, free, activate, deactivate, translate, xlate, map, unmap }
```
Hierarchical domains use `.alloc/.free/.activate/.deactivate`; legacy linear domains use `.map/.xlate`.

`irqchip_init()` order:
1. `of_irq_init(__irqchip_of_table)` walks DT `interrupt-controller` nodes; init-fns called parent-before-child via DT `interrupt-parent` linkage.
2. `acpi_probe_device_table(irqchip)` walks MADT for ACPI systems.
3. Late drivers register via `platform_irqchip_probe` after the device-model is up.

## Hardening

- IRQCHIP_DECLARE table is `__ro_after_init`; no runtime mutation of the compat→init-fn mapping.
- `irq_domain_ops` vtables live in `__ro_after_init` / `const` data; kCFI verifies indirect dispatch.
- Per-domain `revmap` allocations bounded by `hwirq_max`; refuse `irq_create_mapping(d, hwirq)` for `hwirq >= d->hwirq_max`.
- DT `#interrupt-cells` cell-count validated against driver expectation in `.translate`.
- Per-driver MMIO mapping uses `of_iomap` + `ioremap` — kernel-space only, not userspace-exposed.
- Per-CPU cpuhp callbacks (`gic_starting_cpu`, `gic_dying_cpu`, etc.) re-initialize per-CPU registers on hotplug to defeat stale-CPU-state-after-offline.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `irq_domain`, `irq_chip`, `irq_data`, per-chip data slabs whitelisted; any `/proc/interrupts` and `/proc/irq/*` user copies strictly bounded.
- **PAX_KERNEXEC** — irqchip core text W^X; `irq_domain_ops` and `irq_chip` vtables sit in `__ro_after_init` text, not patchable post-boot.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across irq entry / `generic_handle_domain_irq` so per-IRQ stack-relative gadgets cannot be pre-aligned.
- **PAX_REFCOUNT** — saturating `refcount_t` on `irq_domain`, per-mapping refs, and per-msi-desc reference chains; overflow trap defeats irq-detach races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-chip slabs and revmap pages so retired hwirq metadata cannot leak.
- **PAX_UDEREF** — SMAP/PAN on every `/proc/irq/*_affinity_list` and `irq_setaffinity` write that copies from user.
- **PAX_RAP / kCFI** — `irq_domain_ops` (`.alloc`, `.free`, `.translate`, `.activate`, `.deactivate`, `.match`) and `irq_chip` callbacks (`.irq_mask`, `.irq_unmask`, `.irq_eoi`, `.irq_set_type`, `.irq_set_affinity`) are kCFI-typed; indirect dispatch checks the function-type tag before branching.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of irqchip symbols, root-chip handler addresses, per-domain `host_data` pointers behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict irqchip init banners and `irq_domain_*` warnings to CAP_SYSLOG so attackers cannot enumerate the IRQ topology from boot dmesg.
- **IRQCHIP_DECLARE table RO** — `__irqchip_of_table` placed in `.init.data` and discarded after `irqchip_init`; the compat→init-fn binding cannot be mutated to redirect a future re-probe.
- **irq_domain_ops strict kCFI** — every domain-ops dispatch site requires the callee's CFI tag to match the trait shape; refusing untagged or mistagged callees defeats irq-domain-pointer-overwrite ROP.
- **`/proc/irq/N/smp_affinity` CAP_SYS_NICE/CAP_SYS_ADMIN** — affinity writes capability-checked and rate-limited; defense against unprivileged IRQ-steering DoS.

Rationale: irqchip drivers sit on the very first path the CPU takes after a hardware interrupt fires — anything that hijacks `irq_domain_ops` or the root-chip handler pointer bypasses every later protection. kCFI on domain-ops + chip-ops, RO IRQCHIP_DECLARE table, refcount-saturation on per-mapping refs, and capability-gated affinity writes turn the irqchip surface from "trusted by construction" into "structurally enforced".

## Open Questions

- (none at Tier-2 level)

## Out of Scope

- ARM GIC v1/v2 details → `irq-gic.md`
- ARM GICv3 + ITS details → `irq-gic-v3.md`
- MIPS GIC details → `irq-mips-gic.md`
- ARM GICv5 (still settling upstream) → future Tier-3
- RISC-V PLIC/APLIC/IMSIC → future Tier-3
- Apple AIC → future Tier-3
- x86 IO-APIC (lives under `arch/x86/`) → covered by `arch/x86/idt.md`
