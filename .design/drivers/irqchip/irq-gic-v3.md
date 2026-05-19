# Tier-3: drivers/irqchip/irq-gic-v3.c + irq-gic-v3-its.c — ARM GICv3/v4 (Redistributors + LPI + ITS + vSGI)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/irqchip/00-overview.md
upstream-paths:
  - drivers/irqchip/irq-gic-v3.c
  - drivers/irqchip/irq-gic-v3-its.c
  - drivers/irqchip/irq-gic-v3-its-msi-parent.c
  - drivers/irqchip/irq-gic-v3-mbi.c
  - drivers/irqchip/irq-gic-v4.c
  - drivers/irqchip/irq-gic-common.c
  - include/linux/irqchip/arm-gic-v3.h
  - include/linux/irqchip/arm-gic-v4.h
-->

## Summary

ARM Generic Interrupt Controller v3/v4 — the modern ARM64 interrupt architecture. The hardware splits into one Distributor (GICD, MMIO), one Redistributor per CPU (GICR, MMIO, contains per-CPU SGI/PPI control + LPI configuration + LPI pending tables), and one or more Interrupt Translation Service units (ITS, MMIO + DMA command queue + in-memory tables) for message-signalled interrupts. CPU Interface lives in CPU system registers (`ICC_*_EL1`), replacing the MMIO GICC of v2.

GICv4 adds Locality-specific Peripheral Interrupts (LPIs) targeting virtual machines directly via ITS-managed vPE tables (vMOVI/MAPI/etc), enabling KVM to bypass the host on guest-targeted device MSIs. GICv4.1 adds direct vSGI injection (no host trap) for SMP-guest IPI bypass and per-vPE doorbells.

This Tier-3 covers `irq-gic-v3.c` (~2600 lines: distributor + redistributor + sysreg CPU iface init, irq_chip ops, EL2/EL3 entry, NMI plumbing, pseudo-NMI priorities), `irq-gic-v3-its.c` (~5900 lines: ITS init, command queue, device-table + collection-table + interrupt-translation-table, LPI prop/pend tables, vLPI plumbing), `irq-gic-v3-mbi.c` (Message-Based Interrupts), and `irq-gic-v4.c` (vPE handling glue).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct gic_chip_data` | per-instance distributor info + rdists + flags | `drivers::irqchip::gicv3::Gic` |
| `struct rdists` | per-instance redistributor + LPI config table + GICR base array | `Gic::Rdists` |
| `struct its_node` | per-ITS control block (base, cmd_base, tables[], collections, lock) | `Its::Node` |
| `struct its_device` | per-DeviceID record (ITT pointer, event LPI map, vlpi maps) | `Its::Device` |
| `gic_of_init(node, parent)` / `gic_acpi_init(header, end)` | per-bootmodel init | `Gic::of_init` / `Gic::acpi_init` |
| `gic_handle_irq(regs)` | arch-entry root handler reading ICC_IAR1_EL1 | `Gic::handle_irq` |
| `__gic_handle_irq(irqnr, regs)` | inner dispatch: SGI/PPI/SPI/LPI | `Gic::handle_irq_inner` |
| `gic_enable_redist(enable)` | per-CPU GICR enable handshake (clear ProcessorSleep, wait ChildrenAsleep) | `Gic::enable_redist` |
| `gic_compute_target_list(base_cpu, mask, cluster_id)` / `gic_ipi_send_mask` | ICC_SGI1R_EL1 SGI target encoding | `Gic::compute_target_list` / `Gic::ipi_send` |
| `gic_set_affinity(d, mask, force)` | write GICD_IROUTER (or GICR for PPI) — affinity-route | `Gic::set_affinity` |
| `its_init(handle, rdists, parent, prio_irq)` | per-domain ITS subsystem init | `Its::init` |
| `its_send_single_command(its, builder, desc)` | build + enqueue + wait one ITS command | `Its::send_single_command` |
| `its_build_mapd_cmd / _mapc_cmd / _mapti_cmd / _movi_cmd / _discard_cmd / _inv_cmd / _invall_cmd / _vmapp_cmd / _vmovp_cmd / _vinvall_cmd` | per-command builders | `Its::Cmd::*` |
| `its_alloc_device_irq(dev, nvecs, hwirq)` | per-Device allocation of N LPI hwirqs | `Its::alloc_device_irq` |
| `its_allocate_prop_table / _allocate_pending_table` | host LPI configuration tables | `Its::alloc_prop_table` / `_alloc_pending_table` |
| `its_vlpi_map / its_vlpi_unmap / its_vlpi_prop_update` | vLPI binding for GICv4 guest pass-through | `Its::vlpi_*` |
| `its_send_vmapp / _vmapti / _vmovp` | per-vPE collection + per-vLPI map commands | `Its::vmapp` / `_vmapti` / `_vmovp` |
| `gicv4_init / gicv4_sync_hw_state / gicv4_doorbell` | v4/v4.1 host-side helpers | `GicV4::*` |
| `IRQCHIP_DECLARE(gic_v3, "arm,gic-v3", gic_of_init)` | DT registration | `Gic::compat` |

## Compatibility contract

REQ-1: GICD MMIO region at the first `reg` cell; size at least 0x10000. GICR per-CPU regions discovered via `redistributor-regions` count + `reg` cells after distributor; alternatively GICR_TYPER chain-walked from a single base.

REQ-2: CPU interface is system-register based — `ICC_SRE_EL1.SRE=1` enables sysreg mode; `ICC_PMR_EL1` sets priority mask; `ICC_CTLR_EL1.EOImode` selects single vs split EOI; `ICC_IGRPEN1_EL1` enables group-1 NS interrupts.

REQ-3: Interrupt classes: SGI (0–15), PPI (16–31), SPI (32 .. GICD_TYPER.ITLinesNumber*32 − 1), ESPI (extended SPI, 4096–5119 when GICD_TYPER.ESPI), LPI (8192 .. GICD_TYPER.IDbits-bounded); LPIs delivered via Redistributor + ITS, NOT distributor.

REQ-4: Per-CPU Redistributor sequence: `gic_enable_redist(true)` clears GICR_WAKER.ProcessorSleep and polls GICR_WAKER.ChildrenAsleep=0; required before per-CPU PPI/SGI access.

REQ-5: SGI routing via ICC_SGI1R_EL1 sysreg write — affinity-encoded (Aff3.Aff2.Aff1) + target-list bitmap of up to 16 CPUs sharing the same Aff0 high-nibble; cross-cluster IPI requires multiple sysreg writes.

REQ-6: SPI affinity programmed via GICD_IROUTER — 64-bit register per SPI; bit-30 selects "any-CPU" mode, else `MPIDR_EL1[Aff3.Aff2.Aff1.Aff0]` selects exact CPU.

REQ-7: ITS command queue is a circular DMA-coherent buffer; commands are 32-byte aligned; CWRITER register advances tail; HW completes when CREADR catches up. `SYNC` and `INVALL` commands synchronize with host visibility.

REQ-8: ITS tables (Device, Collection, Virtual Collection, Vpe, ITT) allocated in host memory; their physical addresses programmed into GITS_BASERn registers; ITT (per-Device Interrupt Translation Table) allocated lazily on first DeviceID alloc.

REQ-9: LPI configuration table (1 byte per LPI, holds priority + enable bit) shared across CPUs; LPI pending table per-CPU (1 bit per LPI); both allocated in normal memory, addresses programmed into GICR_PROPBASER + GICR_PENDBASER.

REQ-10: GICv4 vPE direct-injection: per-VM `its_vm` allocates `its_vpe_irq_domain`; each guest vCPU gets a `vpe_id`; `VMAPP` command binds vpe_id to host pCPU's Redistributor; `VMAPTI` binds (DeviceID, EventID) → (vpe_id, vLPI); host trap bypassed for matching MSIs.

REQ-11: GICv4.1 vSGI: per-VM 16 vSGIs per vPE; doorbell IRQ per-vPE for non-resident wakeups; ICC_SGI1R-equivalent vSGI injection from guest never traps.

REQ-12: Pseudo-NMI support: priority < `dist_prio_irq` is a pseudo-NMI delivered with DAIF.IRQ unchanged, masked only by PSTATE.PMR; allows perf interrupts to preempt critical sections (under `CONFIG_ARM64_PSEUDO_NMI`).

REQ-13: MBI (Message-Based Interrupts) `gic-v3-mbi.c`: alternative MSI source via GICD_SETSPI_NSR write; used when ITS is absent.

REQ-14: Errata workarounds via `gic_quirks[]` table — Cavium ThunderX (TYPER2 explodes on read), NVIDIA T241-FABRIC-4, HiSilicon hip06/hip07, Marvell 38539, etc.

## Acceptance Criteria

- [ ] AC-1: Boot on ARM64 QEMU virt (`-machine virt,gic-version=3`) — dmesg shows distributor + N redistributors discovered + LPI table allocation.
- [ ] AC-2: Boot on hardware with ITS (Ampere Altra, AWS Graviton, Apple Mx via Asahi adjusted, NXP LX2160) — `dmesg | grep ITS` shows ITS init + device-table + collection-table allocated, ITS CTLR.Enabled=1.
- [ ] AC-3: PCIe NVMe MSI-X allocation succeeds — `cat /proc/interrupts | grep nvme` shows LPI hwirqs ≥ 8192; affinity writes succeed.
- [ ] AC-4: KVM-VGICv3 guest with virtio-net passes traffic; `kvm-unit-tests/arm/gicv3-ipi` passes.
- [ ] AC-5: GICv4 platform: KVM PCIe passthrough with `kvm-arm.vgic_v4_enable=1` — `dmesg | grep vLPI` shows vPE-bind succeeded; guest MSI direct-injects observable via reduced exit-count.
- [ ] AC-6: Pseudo-NMI test: `CONFIG_ARM64_PSEUDO_NMI=y` + perf running — PMU IRQ delivered as NMI; `lockdep` doesn't complain about NMI in held-spinlock context.
- [ ] AC-7: CPU hotplug stress cycles 1000 iterations on 8-way GICv3 system — redistributor enable/disable handshake completes; no LPI delivery loss.

## Architecture

`Gic` (Rust analogue):

```
struct Gic {
  dist_base: NonNull<u8>,
  redist_regions: Vec<RedistRegion>,
  redist_stride: u64,
  rdists: Rdists,          // includes per-CPU rdist base, has_vlpis, has_rvpeid, prop_table
  domain: Arc<IrqDomain>,
  ppi_partitions: Vec<PartitionAffinity>,
  has_rss: bool,            // GICD_TYPER.RSS = range-selector for SGI > 16 cpus per cluster
  has_mbis: bool,
  flags: GicQuirkFlags,
}
```

`Its::Node`:

```
struct Node {
  base:        NonNull<u8>,
  sgir_base:   Option<NonNull<u8>>,   // GICv4.1 vSGI MMIO window
  phys_base:   PhysAddr,
  cmd_base:    NonNull<CmdBlock>,
  cmd_write:   AtomicPtr<CmdBlock>,
  tables:      [Baser; GITS_BASER_NR_REGS],   // device / collection / vpe / vcoll
  collections: NonNull<Collection>,           // per-CPU collection
  fwnode:      Arc<FwnodeHandle>,
  typer:       u64,
  numa_node:   NumaNode,
  vlpi_redist_offset: u32,
  device_list: Mutex<Vec<Arc<Device>>>,
  lock: RawSpinLock,
  dev_alloc_lock: Mutex<()>,
}
```

Init flow `gic_of_init`:
1. Memremap distributor base; validate `GICD_PIDR2.ARCH ∈ {GICv3, GICv4}`.
2. Discover per-CPU redistributors: either iterate per `redistributor-regions` cells or chain-walk via `GICR_TYPER.Last`.
3. Read GICD_TYPER → compute SPI count, ESPI count, IDbits.
4. `irq_domain_create_tree(handle, &gic_irq_domain_ops, gic_data)`.
5. Set rdists capability flags from MPIDR + GICR_TYPER: `has_rvpeid`, `has_vlpis`, `has_direct_lpi`, `has_vpend_valid_dirty`.
6. `set_handle_irq(gic_handle_irq)`.
7. `gic_dist_init()` → write GICD_IGROUPRn=all-Group-1, GICD_IPRIORITYRn=dist_prio_irq, GICD_ICFGRn=level, GICD_CTLR.EnableGrp1NS=1 (+ ARE bit).
8. `gic_cpu_sys_reg_enable()` → ICC_SRE_EL1.SRE=1.
9. `gic_prio_init()` → ICC_PMR_EL1=GIC_PRIO_IRQON.
10. `gic_cpu_init()` → enable PPIs, ICC_CTLR_EL1 setup, ICC_IGRPEN1_EL1=1.
11. `its_init(handle, &rdists, domain, dist_prio_irq)` if `GICD_TYPER.LPIS=1`.
12. `its_cpu_init()` per-CPU → program GICR_PROPBASER + GICR_PENDBASER + clear GICR_WAKER.

Hot-path IRQ dispatch `__gic_handle_irq(irqnr, regs)`:
- irqnr 0..15 → SGI → `generic_handle_domain_irq(ipi_domain, irqnr)`.
- 16..1019 → SPI/PPI → `generic_handle_domain_irq(gic->domain, irqnr)`.
- ≥ 8192 → LPI → `generic_handle_domain_irq(gic->domain, irqnr)`.
- 1023 → spurious; ignore.
EOI via `gic_write_eoir(irqnr)` (or split-mode: priority drop then `gic_write_dir`).

ITS command path `its_send_single_command(its, builder, desc)`:
1. `raw_spin_lock_irqsave(&its->lock)`.
2. `its_allocate_entry(its)` advances `cmd_write` (wrap on `(cmd_base + ITS_QUEUE_SIZE)`).
3. `builder(its, cmd, desc)` fills the 32-byte command (e.g. `MAPD` with device_id + ITT_addr + valid bit).
4. Append SYNC command bound to target collection (cross-CPU visibility fence).
5. `writel_relaxed(cmd_write_offset, its->base + GITS_CWRITER)`.
6. Spin polling `GITS_CREADR` until ≥ cmd_write_offset OR sleep on completion var.
7. Unlock.

vLPI binding `its_vlpi_map(map_info)`:
1. Allocate vPE-id via `its_vpeid_ida` (or use existing `vpe_proxy` for GICv4.0 indirect mode).
2. `its_send_vmapp(its, vpe, true)` — bind vpe_id to per-CPU host pCPU's redistributor + virt-pending-table.
3. `its_send_vmapti(dev, event_id, vpe_id, vlpi)` — bind (DeviceID, EventID) → (vpe_id, vLPI).
4. Send `VINVALL` to flush per-vCPU LPI config cache.

Per-PPI partitions (used by big.LITTLE perf): each PPI may have multiple partition-IDs each affined to a subset of CPUs; resolved at `irq_domain_alloc` time via `partition_translate_id` lookup.

GICv4.1 vSGI doorbell: per-vPE LPI configured as "doorbell"; signals when a vSGI arrives for a non-resident vPE; host scheduler then resumes the vCPU.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `its_cmd_queue_wrap` | OOB | `cmd_write` wraps at `ITS_QUEUE_SIZE`; index always within mapped range. |
| `lpi_alloc_no_dup` | UNIQUENESS | per-ITS LPI bitmap ensures no LPI hwirq allocated twice. |
| `vpe_id_bounds` | OOB | vPE-id ≤ `(1 << GICD_TYPER2.VIL)`; rejected otherwise. |
| `redist_handshake_terminates` | LIVENESS | enable_redist polling bounded; timeout aborts boot with WARN. |
| `affinity_in_mpidr_format` | INVARIANT | GICD_IROUTER write equals `mpidr_to_affinity_level(cpu_mpidr)`. |

### Layer 2: TLA+

`models/irqchip/gicv3_its_cmd.tla` (future): models concurrent producers + per-ITS lock + single consumer (HW) + SYNC fence; proves command serialization observed by HW matches order in cmd_base.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `its_send_mapd` post: device-table entry for `device_id` has `valid=1`, ITT-addr matches allocated phys | `Its::send_mapd` |
| `its_send_discard` post: per-(device,event) ITTE.valid=0; LPI bitmap bit cleared | `Its::send_discard` |
| `gic_set_affinity` post: GICD_IROUTER reads back exactly the encoded MPIDR | `Gic::set_affinity` |
| `vlpi_map` post: per-vPE-table vMAP_C entry contains target host pCPU rdist offset | `Its::vlpi_map` |

### Layer 4: Verus/Creusot functional

Encoded: an MSI write from a PCIe device with translated (DeviceID, EventID) reaches the ITS, which translates to LPI L, fires on the assigned CPU's redistributor, which delivers to the handler bound via `irq_domain_alloc`. End-to-end safety chained with `drivers/pci/00-overview.md` MSI write path.

## Hardening

- ITS command queue size bounded (default 64 KB); per-command lock-step builder + tail-advance prevents partial commands.
- LPI prop/pending tables sized to GICD_TYPER.IDbits with `gic_reserve_range` against memblock; refuses to allocate when IDbits exceeds host-supportable range.
- Per-ITS device-list rwlock + `dev_alloc_lock` mutex; device-table entries written under both.
- GICv4 vPE-id allocator uses `ida_alloc`; bounded by GICD_TYPER2.VIL; can't outgrow HW capability.
- Quirks table (`gic_quirks[]`) consulted at init for known-bad GICs; per-quirk action either disables a feature or reroutes accesses.
- Redistributor wake handshake has `timeout=USEC_PER_SEC` ceiling; refuses to proceed past timeout, surfaces in `dmesg`.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `gic_chip_data`, `its_node`, `its_device`, `its_vm`, `its_vpe` slab caches whitelisted; vLPI pending-table user-mapped pages refuse mmap from userspace.
- **PAX_KERNEXEC** — irqchip core text W^X; `gic_irq_domain_ops`, `its_irq_domain_ops`, `gic_chip` callbacks, and command-builder dispatch in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `gic_handle_irq`, `__gic_handle_irq`, ITS-IRQ handler, and pseudo-NMI entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on `its_node` membership, `its_device` per-event refs, `its_vm` per-vCPU refs, and per-vPE collection refs; overflow trap defeats vLPI unbind races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for ITT pages, LPI prop/pend tables, per-vPE pending tables, and command-queue pages so retired (DeviceID,EventID) mappings don't bleed to a successor allocation.
- **PAX_UDEREF** — SMAP/PAN on every iommufd / KVM ioctl path that flows into ITS device alloc; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `gic_chip` ops, `its_irq_domain_ops`, `its_pmsi_domain_ops`, ITS command builders (`its_build_*_cmd`), and `gic_pm_ops` callbacks marked kCFI-typed.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `gic_data`, `gic_rdists`, `its_nodes` list, per-ITS base pointers, and vPE-id allocator state behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict GICv3/ITS init banners, redistributor wake-timeouts, and vLPI bind failures to CAP_SYSLOG so attackers cannot enumerate the LPI topology or ITS device-table layout via dmesg.
- **ITS command-queue PAX_USERCOPY** — command-queue pages whitelisted with strict bounds; no userspace mmap path exists, and KVM-ITS migration uses dedicated kernel→kernel copies validated against `cmd_base` extent.
- **Device-table / collection-table PAX_REFCOUNT** — per-DeviceID and per-collection refs trapping on overflow; defense against double-MAPD and double-MAPC race UAFs.
- **LPI prop / pend / config tables RO** — prop_table and pend_table base addresses written once at init and locked; refuse re-program of GICR_PROPBASER / GICR_PENDBASER post-`gic_starting_cpu`.
- **vSGI gating CAP_SYS_ADMIN** — KVM ioctls that enable GICv4.1 vSGI direct-injection require CAP_SYS_ADMIN in the VM owner's user-ns; per-VM vPE-id allocation refused otherwise.
- **ITS device-id space validated** — `nvecs` bounded by `2 ^ device_ids(its)`; refuse DeviceID outside the HW-declared range; defense against ITS-table-overflow leading to OOB write.
- **Pseudo-NMI priority strict** — `dist_prio_irq` set at init from a fixed band and `__ro_after_init`; refuse PMR writes that would let arbitrary IRQs masquerade as pseudo-NMIs.

Rationale: GICv3+ITS is the authoritative arbiter of every MSI-X delivery on ARM64 servers and the direct-injection bypass for KVM guests on GICv4. A corrupted ITS command, a stale vPE-id, an unbounded device-id, or a vSGI mis-route silently steers physical or virtualized interrupts to attacker-chosen CPUs. kCFI on chip-ops + ITS-builders, refcount saturation on device/vPE refs, RO LPI tables post-init, and CAP_SYS_ADMIN-gated vSGI enablement turn GICv3/v4 from "isolation if microcode behaves" into a structural enforcement boundary.

## Open Questions

- (none at this Tier-3 level)

## Out of Scope

- ARM GICv5 IRS/ITS/IWB (still settling upstream) — future Tier-3
- ARM GIC v2 + v2m (covered in `irq-gic.md`)
- KVM-VGICv3 internals (covered in `virt/kvm/00-overview.md`)
- ARM SMMU + MSI passthrough integration (covered in `drivers/iommu/00-overview.md` future arm-smmu Tier-3)
- 32-bit ARM GICv3 ports (rare; not the focus)
- Implementation code
