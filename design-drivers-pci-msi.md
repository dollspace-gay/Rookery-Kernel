---
title: "Tier-3: drivers/pci/msi/msi.c — PCI MSI / MSI-X allocation"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-11
updated: 2026-05-11
---


## Design Specification

### Summary

Message Signaled Interrupts (MSI) replace the legacy INTx wire interrupts with memory writes by the device to a CPU-readable address with a vector-encoding payload. **MSI** uses a single Capability (PCI_CAP_ID_MSI = 0x05) with 1, 2, 4, 8, 16, or 32 contiguous vectors and an in-config-space mask-bit register. **MSI-X** uses Capability (PCI_CAP_ID_MSIX = 0x11) plus an MMIO Vector Table (in one of the device BARs) of up to 2048 independent vectors, each with its own address/data/control triple. Per-device allocation flow: `pci_alloc_irq_vectors(dev, min, max, flags)` ⟶ tries MSI-X then MSI then INTx ⟶ on success returns the number of allocated vectors, and `pci_irq_vector(dev, nr)` resolves to a Linux IRQ number suitable for `request_threaded_irq`. Under the hood, MSI/MSI-X capability is set up via `msi_capability_init` / `msix_capability_init` which allocates `struct msi_desc`s under the hierarchical irq_domain (`pci_msi_template` / `pci_msix_template` with `struct irq_chip` ops `pci_irq_mask_msi` / `pci_irq_unmask_msi` / `pci_msi_domain_write_msg`) and programs the device MSI/MSI-X capability registers (address-lo/-hi, data, mask, ENABLE bit).

This Tier-3 covers `drivers/pci/msi/msi.c` (~993 lines) plus the per-device irq_domain template wired in via `drivers/pci/msi/irqdomain.c` and the disable-on-probe stubs in `drivers/pci/msi/pcidev_msi.c`.

### Acceptance Criteria

- [ ] AC-1: pci_alloc_irq_vectors(dev, 1, N, PCI_IRQ_MSIX|PCI_IRQ_MSI|PCI_IRQ_INTX) on MSI-X-capable device returns N (≤ dev.msix_cap table-size) and sets dev.msix_enabled = 1.
- [ ] AC-2: pci_alloc_irq_vectors on MSI-only device (no msix_cap) falls back to MSI and returns a power-of-2 ≤ msi_vec_count(dev).
- [ ] AC-3: pci_alloc_irq_vectors with PCI_IRQ_MSIX-only on device with msix_cap=0 returns 0 (no MSI-X); does not silently fall through.
- [ ] AC-4: pci_free_irq_vectors(dev) clears dev.msi_enabled, dev.msix_enabled; iounmaps msix_base; releases all msi_descs.
- [ ] AC-5: pci_irq_vector(dev, idx) returns the Linux IRQ number for index idx ∈ 0..N-1; -EINVAL for out-of-range.
- [ ] AC-6: MSI capability ENABLE bit set in config space after capability_init returns 0; cleared after shutdown_msi.
- [ ] AC-7: MSI-X ENABLE bit + MASKALL bit pattern follows spec: MASKALL set during table setup; cleared once vectors are programmed and unmasked individually.
- [ ] AC-8: __pci_write_msi_msg writes address_lo/hi/data to MSI cap (config space) for MSI, or to MSI-X table (MMIO) for MSI-X.
- [ ] AC-9: Per-vector mask via pci_irq_mask_msi sets bit in msi_mask register (config) and is reflected in subsequent reads; symmetric for pci_irq_unmask_msi.
- [ ] AC-10: Per-vector MSI-X mask via pci_irq_mask_msix writes PCI_MSIX_ENTRY_CTRL_MASKBIT to the table entry's VECTOR_CTRL word.
- [ ] AC-11: pci_msi_supported denies MSI when bus_flags has PCI_BUS_FLAGS_NO_MSI on any ancestor bus.
- [ ] AC-12: PM resume: __pci_restore_msi_state rewrites entry.msg + control word with stored QSIZE; __pci_restore_msix_state rewrites every table entry's addr/data/ctrl from the cached msi_desc + msix_ctrl.
- [ ] AC-13: Multi-MSI: only valid power-of-2 nvec ∈ {1,2,4,8,16,32} ≤ multi_cap; capability_init returns 1 on domain that does not support multi-MSI.
- [ ] AC-14: pci_no_msi() globally disables MSI; subsequent pci_alloc_irq_vectors with PCI_IRQ_MSIX|PCI_IRQ_MSI falls through to INTx.
- [ ] AC-15: msi_verify_entries rejects any vector with address > dev.msi_addr_mask with -EIO (defense against arch assigning 64-bit MSI address to 32-bit MSI device).

### Architecture

```
struct PciMsiState {                     // embedded in PciDev
  msi_cap: u8,                           // offset of MSI cap (or 0)
  msix_cap: u8,                          // offset of MSI-X cap (or 0)
  msi_enabled: bool,
  msix_enabled: bool,
  msix_base: Option<*Iomem>,             // ioremap of MSI-X table
  msi_addr_mask: u64,                    // DMA mask for MSI msg addrs
  msi_lock: RawSpinLock,                 // mask RMW
  is_msi_managed: bool,                  // pcim auto-release flag
  no_msi: bool,                          // quirk
}

struct MsiDesc {
  dev: *Device,                          // == &pci_dev.dev
  irq: u32,                              // Linux IRQ (base for multi-MSI)
  nvec_used: u32,
  msg: MsiMsg,
  affinity: Option<*IrqAffinityDesc>,
  msi_index: u32,
  pci: {
    mask_pos: u32,                       // config offset of MSI mask reg
    msi_mask: u32,                       // cached MSI mask
    mask_base: Option<*Iomem>,           // MSI-X table base
    msix_ctrl: u32,                      // cached MSI-X vector ctrl
    msi_attrib: {
      is_msix: bool,
      is_64: bool,
      is_virtual: bool,
      can_mask: bool,
      default_irq: u32,
      multi_cap: u8,                     // log2 max vectors
      multiple: u8,                      // log2 requested
    },
  },
}

struct MsiMsg {
  address_lo: u32,
  address_hi: u32,
  data: u32,
}

struct MsixEntry {                       // caller-supplied
  vector: u32,                           // out: Linux IRQ
  entry: u16,                            // in: table index
}
```

`PciMsi::alloc_irq_vectors(dev, min_vecs, max_vecs, flags) -> Result<i32, errno>`:
1. Try MSI-X (if flags & PCI_IRQ_MSIX):
   - nvecs = PciMsi::enable_msix_range(dev, NULL, min, max, NULL, flags).
   - if nvecs > 0: return Ok(nvecs).
2. Try MSI (if flags & PCI_IRQ_MSI):
   - nvecs = PciMsi::enable_msi_range(dev, min, max, NULL).
   - if nvecs > 0: return Ok(nvecs).
3. INTx fallback (if flags & PCI_IRQ_INTX ∧ min == 1 ∧ dev.irq != 0):
   - pci_intx(dev, 1); return Ok(1).
4. Err.

`PciMsi::capability_init(dev, nvec, affd) -> i32`:
1. /* Reject multi-MSI on non-supporting domain */
2. if nvec > 1 ∧ !pci_msi_domain_supports(dev, MSI_FLAG_MULTI_PCI_MSI, ALLOW_LEGACY): return 1.
3. /* Hardware-disable MSI during setup */
4. PciMsi::set_enable(dev, 0).
5. masks = affd ? irq_create_affinity_masks(nvec, affd) : NULL.
6. guard(msi_descs_lock)(&dev.dev).
7. ret = PciMsi::setup_msi_desc(dev, nvec, masks).
8. if ret != 0: return ret.
9. entry = msi_first_desc(&dev.dev, MSI_DESC_ALL).
10. pci_msi_mask(entry, msi_multi_mask(entry)) /* mask all before enable */.
11. desc_copy = *entry.
12. ret = pci_msi_setup_msi_irqs(dev, nvec, PCI_CAP_ID_MSI).
13. if ret != 0: goto err.
14. ret = PciMsi::verify_entries(dev).
15. if ret != 0: goto err.
16. dev.msi_enabled = true.
17. pci_intx_for_msi(dev, 0).
18. PciMsi::set_enable(dev, 1).
19. pcibios_free_irq(dev).
20. dev.irq = entry.irq.
21. return 0.
22. err:
    - pci_msi_unmask(&desc_copy, msi_multi_mask(&desc_copy)).
    - pci_free_msi_irqs(dev).
    - return ret.

`PciMsi::msix_capability_init(dev, entries, nvec, affd) -> i32`:
1. /* Enable MSI-X + mask-all before setup to prevent stray interrupts */
2. PciMsi::msix_clear_and_set_ctrl(dev, 0, PCI_MSIX_FLAGS_MASKALL | PCI_MSIX_FLAGS_ENABLE).
3. dev.msix_enabled = true /* allow setup queries */.
4. ctrl = config_read_u16(dev, msix_cap + PCI_MSIX_FLAGS).
5. tsize = (ctrl & PCI_MSIX_FLAGS_QSIZE) + 1.
6. dev.msix_base = PciMsi::msix_map_region(dev, tsize).
7. if dev.msix_base.is_none(): ret = -ENOMEM; goto out_disable.
8. ret = PciMsi::msix_setup_interrupts(dev, entries, nvec, affd).
9. if ret != 0: goto out_unmap.
10. pci_intx_for_msi(dev, 0).
11. /* Late mask-all (Marvell-NVMe quirk: cares about table mask bits even when MSI-X disabled) */
12. if !pci_msi_domain_supports(dev, MSI_FLAG_NO_MASK, DENY_LEGACY):
    - PciMsi::msix_mask_all(dev.msix_base, tsize).
13. PciMsi::msix_clear_and_set_ctrl(dev, PCI_MSIX_FLAGS_MASKALL, 0) /* clear all-vectors mask */.
14. pcibios_free_irq(dev).
15. return 0.
16. out_unmap: iounmap(dev.msix_base).
17. out_disable: dev.msix_enabled = false; PciMsi::msix_clear_and_set_ctrl(dev, MASKALL | ENABLE, 0).

`PciMsi::msix_setup_msi_descs(dev, entries, nvec, masks) -> i32`:
1. vec_count = PciMsi::msix_vec_count(dev).
2. for i in 0..nvec:
   - desc = MsiDesc::zeroed.
   - desc.msi_index = entries.is_some() ? entries[i].entry : i.
   - desc.affinity = masks.is_some() ? &masks[i] : NULL.
   - desc.pci.msi_attrib.is_virtual = desc.msi_index >= vec_count.
   - PciMsi::prepare_msix_desc(dev, &desc).
   - if msi_insert_msi_desc(&dev.dev, &desc).is_err(): break.
3. return ret.

`PciMsi::write_msg(entry, msg)` /* __pci_write_msi_msg */:
1. dev = msi_desc_to_pci_dev(entry).
2. if entry.pci.msi_attrib.is_msix:
   - base = pci_msix_desc_addr(entry).
   - writel(msg.address_lo, base + PCI_MSIX_ENTRY_LOWER_ADDR).
   - writel(msg.address_hi, base + PCI_MSIX_ENTRY_UPPER_ADDR).
   - writel(msg.data, base + PCI_MSIX_ENTRY_DATA).
3. else /* MSI */:
   - pos = dev.msi_cap.
   - config_write_u32(dev, pos + PCI_MSI_ADDRESS_LO, msg.address_lo).
   - if entry.pci.msi_attrib.is_64:
     - config_write_u32(dev, pos + PCI_MSI_ADDRESS_HI, msg.address_hi).
     - config_write_u16(dev, pos + PCI_MSI_DATA_64, msg.data as u16).
   - else:
     - config_write_u16(dev, pos + PCI_MSI_DATA_32, msg.data as u16).
4. entry.msg = *msg /* cache for restore */.

`PciMsi::msi_chip_mask(data)` /* irq_chip MSI mask */:
1. desc = irq_data_get_msi_desc(data).
2. pci_msi_mask(desc, BIT(data.irq - desc.irq)) /* sets bit in desc.pci.msi_mask, writes to config */.

`PciMsi::msix_chip_mask(data)` /* irq_chip MSI-X mask */:
1. desc = irq_data_get_msi_desc(data).
2. addr = pci_msix_desc_addr(desc).
3. ctrl = desc.pci.msix_ctrl | PCI_MSIX_ENTRY_CTRL_MASKBIT.
4. desc.pci.msix_ctrl = ctrl.
5. writel(ctrl, addr + PCI_MSIX_ENTRY_VECTOR_CTRL).

`PciMsi::set_enable(dev, en)`:
1. ctrl = config_read_u16(dev, dev.msi_cap + PCI_MSI_FLAGS).
2. ctrl &= ~PCI_MSI_FLAGS_ENABLE.
3. if en: ctrl |= PCI_MSI_FLAGS_ENABLE.
4. config_write_u16(dev, dev.msi_cap + PCI_MSI_FLAGS, ctrl).

`PciMsi::msix_clear_and_set_ctrl(dev, clear, set)`:
1. ctrl = config_read_u16(dev, dev.msix_cap + PCI_MSIX_FLAGS).
2. ctrl = (ctrl & ~clear) | set.
3. config_write_u16(dev, dev.msix_cap + PCI_MSIX_FLAGS, ctrl).

irq_chip ops are bound via `msi_domain_template`:
- `pci_msi_template`: chip = { name: "PCI-MSI", irq_mask: PciMsi::msi_chip_mask, irq_unmask: PciMsi::msi_chip_unmask, irq_write_msi_msg: PciMsi::domain_write_msg, irq_startup: PciMsi::msi_startup, irq_shutdown: PciMsi::msi_shutdown, flags: IRQCHIP_ONESHOT_SAFE }; info.flags = MSI_COMMON_FLAGS | MSI_FLAG_MULTI_PCI_MSI; bus_token = DOMAIN_BUS_PCI_DEVICE_MSI.
- `pci_msix_template`: chip = { name: "PCI-MSIX", irq_mask: PciMsi::msix_chip_mask, ... }; ops.prepare_desc = PciMsi::prepare_msix_desc; info.flags = MSI_COMMON_FLAGS | MSI_FLAG_PCI_MSIX | MSI_FLAG_PCI_MSIX_ALLOC_DYN; bus_token = DOMAIN_BUS_PCI_DEVICE_MSIX.

### Out of Scope

- Legacy non-hierarchical MSI path (`pci_msi_legacy_setup_msi_irqs`) — deferred; modern arches use hierarchical irq_domains
- IRQ affinity-spreading algorithm (`irq_create_affinity_masks`, `irq_calc_affinity_vectors`) — covered in `kernel/irq/affinity.md` future Tier-3
- struct irq_domain hierarchical core (`msi_domain_alloc_irqs_all_locked`) — covered in `kernel/irq/msi.md` future Tier-3
- Per-architecture MSI domain backends (x86 APIC, ARM GICv3/v3+ITS, RISC-V IMSIC) — covered in respective arch Tier-3
- TPH (TLP Processing Hints) — `pci_msix_write_tph_tag` is a thin wrapper; full TPH cap covered in `pcie-tph.md` future Tier-3
- ACPI / OF MSI-domain lookup (`pci_msi_get_device_domain`) — covered in `pci-acpi.md` / `pci-of.md` future Tier-3
- pcim_enable_device managed-IRQ side-effect (legacy) — deferred
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `pci_msi_enable` | global MSI on/off flag | shared |
| `pci_alloc_irq_vectors(dev, min, max, flags)` | per-device allocate IRQ vectors (MSI-X / MSI / INTx) | `PciMsi::alloc_irq_vectors` |
| `pci_alloc_irq_vectors_affinity(dev, min, max, flags, affd)` | per-device allocate with affinity-hint | `PciMsi::alloc_irq_vectors_affinity` |
| `pci_free_irq_vectors(dev)` | per-device free + disable MSI/MSI-X | `PciMsi::free_irq_vectors` |
| `pci_irq_vector(dev, nr)` | per-(dev, idx) ⟶ Linux IRQ number | `PciMsi::irq_vector` |
| `pci_irq_get_affinity(dev, nr)` | per-(dev, idx) ⟶ affinity cpumask | `PciMsi::irq_get_affinity` |
| `pci_msi_vec_count(dev)` | per-device max-MSI vector count (capability) | `PciMsi::msi_vec_count` |
| `pci_msix_vec_count(dev)` | per-device max-MSI-X vector count (table-size) | `PciMsi::msix_vec_count` |
| `pci_msi_init(dev)` | per-device discover MSI cap; clear ENABLE | `PciDev::msi_init` |
| `pci_msix_init(dev)` | per-device discover MSI-X cap; clear ENABLE | `PciDev::msix_init` |
| `msi_capability_init(dev, nvec, affd)` | per-device MSI cap setup + allocate | `PciMsi::capability_init` |
| `msix_capability_init(dev, entries, nvec, affd)` | per-device MSI-X cap setup + allocate | `PciMsi::msix_capability_init` |
| `__msi_capability_init(dev, nvec, masks)` | per-device MSI inner (descs + domain alloc + ENABLE) | `PciMsi::capability_init_inner` |
| `__pci_enable_msi_range(dev, minvec, maxvec, affd)` | per-device range-based MSI enable | `PciMsi::enable_msi_range` |
| `__pci_enable_msix_range(dev, entries, minvec, maxvec, affd, flags)` | per-device range-based MSI-X enable | `PciMsi::enable_msix_range` |
| `pci_msi_supported(dev, nvec)` | per-device supports MSI? (global + no_msi + parent NO_MSI bus) | `PciMsi::supported` |
| `pci_msi_set_enable(dev, en)` | per-device write MSI ENABLE bit | `PciMsi::set_enable` |
| `pci_msix_clear_and_set_ctrl(dev, clr, set)` | per-device RMW MSI-X ctrl | `PciMsi::msix_clear_and_set_ctrl` |
| `msix_map_region(dev, n)` | per-device MSI-X table ioremap from BIR | `PciMsi::msix_map_region` |
| `msix_mask_all(base, n)` | per-table mask all vectors | `PciMsi::msix_mask_all` |
| `msi_setup_msi_desc(dev, nvec, masks)` | per-device build MSI msi_desc | `PciMsi::setup_msi_desc` |
| `msix_setup_msi_descs(dev, entries, nvec, masks)` | per-device build per-vector MSI-X msi_descs | `PciMsi::setup_msix_descs` |
| `msix_prepare_msi_desc(dev, desc)` | per-vector half-init msi_desc | `PciMsi::prepare_msix_desc` |
| `msi_verify_entries(dev)` | per-device check addrs ≤ dev.msi_addr_mask | `PciMsi::verify_entries` |
| `pci_msi_update_mask(desc, clear, set)` | per-vector update MSI mask | `PciMsi::update_mask` |
| `pci_msi_mask_irq(data)` | irq_chip mask callback (MSI) | `PciMsi::mask_irq` |
| `pci_msi_unmask_irq(data)` | irq_chip unmask callback (MSI) | `PciMsi::unmask_irq` |
| `__pci_read_msi_msg(entry, msg)` | per-vector read addr/data from config or table | `PciMsi::read_msg` |
| `__pci_write_msi_msg(entry, msg)` | per-vector write addr/data to config or table | `PciMsi::write_msg` |
| `pci_msi_shutdown(dev)` | per-device disable MSI; restore INTx | `PciMsi::shutdown_msi` |
| `pci_msix_shutdown(dev)` | per-device disable MSI-X; restore INTx | `PciMsi::shutdown_msix` |
| `pci_free_msi_irqs(dev)` | per-device teardown + iounmap msix_base | `PciMsi::free_msi_irqs` |
| `__pci_restore_msi_state(dev)` | per-device PM-resume restore MSI | `PciMsi::restore_msi_state` |
| `__pci_restore_msix_state(dev)` | per-device PM-resume restore MSI-X | `PciMsi::restore_msix_state` |
| `pci_msi_setup_msi_irqs(dev, nvec, type)` | hierarchical-domain or legacy-arch IRQ alloc | `PciMsi::setup_msi_irqs` |
| `pci_msi_teardown_msi_irqs(dev)` | hierarchical-domain or legacy-arch IRQ free | `PciMsi::teardown_msi_irqs` |
| `pci_msi_domain_write_msg(irq_data, msg)` | irq_chip write callback (config-space write) | `PciMsi::domain_write_msg` |
| `pci_irq_mask_msi(data)` / `pci_irq_unmask_msi(data)` | per-vector mask via msi_desc mask-bit | `PciMsi::msi_chip_mask` / `_unmask` |
| `pci_irq_mask_msix(data)` / `pci_irq_unmask_msix(data)` | per-vector mask via MMIO ctrl word | `PciMsi::msix_chip_mask` / `_unmask` |
| `pci_msi_template` / `pci_msix_template` | per-msi_domain_template (irq_chip + ops + info) | `PciMsi::msi_template` / `_msix_template` |
| `pci_setup_msi_device_domain(dev, hwsize)` | per-device-MSI irq_domain create | `PciMsi::setup_msi_device_domain` |
| `pci_setup_msix_device_domain(dev, hwsize)` | per-device-MSI-X irq_domain create | `PciMsi::setup_msix_device_domain` |
| `pci_intx_for_msi(dev, en)` | per-device toggle INTx-emulation when MSI active | `PciMsi::intx_for_msi` |
| `msi_desc_to_pci_dev(desc)` | per-desc ⟶ pci_dev | `MsiDesc::to_pci_dev` |
| `pci_no_msi()` | per-system disable MSI globally | `PciMsi::no_msi` |
| `struct msi_desc` | per-vector descriptor (irq, msg, mask, attribs) | `MsiDesc` |
| `struct msi_msg` | per-vector address-lo/hi + data | `MsiMsg` |
| `struct msix_entry` | caller-side (idx, vector) pair | `MsixEntry` |
| `struct irq_chip` | per-irq mask/unmask/write callbacks | `IrqChip` |

### compatibility contract

REQ-1: struct msi_desc (per-vector):
- dev: per-owning struct device (== &pci_dev.dev).
- irq: per-Linux-IRQ-number (base of contiguous range for multi-MSI).
- nvec_used: per-multi-MSI count (1 for MSI-X).
- msg: per-MsiMsg cache (address_hi, address_lo, data).
- affinity: per-irq_affinity_desc array (NULL if not affinity-managed).
- msi_index: per-MSI-X-table-index (or per-MSI-vec-index).
- pci.msi_attrib (bitfield):
  - is_msix: 1 ⟹ MSI-X, 0 ⟹ MSI.
  - is_64: PCI_MSI_FLAGS_64BIT or always 1 for MSI-X.
  - is_virtual: per-virtual-vector beyond hwsize (PCI_IRQ_VIRTUAL flag).
  - can_mask: per-PCI_MSI_FLAGS_MASKBIT (MSI) or always 1 for MSI-X.
  - default_irq: per-restore-target INTx (dev.irq snapshot).
  - multi_cap: per-Multi-Message-Capable field (log2 of max nvec).
  - multiple: per-actually-requested log2.
- pci.mask_pos: per-config-space offset of MSI mask register (msi_cap+PCI_MSI_MASK_32 or _64).
- pci.msi_mask: per-cached MSI mask u32.
- pci.mask_base: per-MSI-X-table ioremap base.
- pci.msix_ctrl: per-MSI-X-vector ctrl word cache.

REQ-2: struct pci_dev MSI fields (populated by msi.c):
- msi_cap: per-MSI-Cap offset (set by pci_msi_init).
- msix_cap: per-MSI-X-Cap offset (set by pci_msix_init).
- msi_enabled: 1 ⟹ MSI active.
- msix_enabled: 1 ⟹ MSI-X active.
- msix_base: per-MSI-X-table ioremap base (NULL if not enabled).
- msi_addr_mask: per-device DMA mask for MSI addresses (default 64-bit, 32-bit if !PCI_MSI_FLAGS_64BIT).
- msi_lock: per-device raw_spinlock_t for mask RMW.
- is_msi_managed: per-pcim_msi_release auto-cleanup flag.
- no_msi: per-quirk forced-off.
- bus.bus_flags & PCI_BUS_FLAGS_NO_MSI: per-bus forced-off (parent bridge can't forward MSI).

REQ-3: PciMsi::supported(dev, nvec) -> bool:
- if !pci_msi_enable: false.
- if dev == NULL ∨ dev.no_msi: false.
- if nvec < 1: false /* nvec >= 1 required */.
- /* Walk bus parents for NO_MSI flag */
- for bus = dev.bus; bus; bus = bus.parent:
  - if bus.bus_flags & PCI_BUS_FLAGS_NO_MSI: false.
- true.

REQ-4: PciMsi::alloc_irq_vectors(dev, min_vecs, max_vecs, flags) -> Result<i32, errno>:
- /* flags: PCI_IRQ_MSIX | PCI_IRQ_MSI | PCI_IRQ_INTX | PCI_IRQ_AFFINITY | PCI_IRQ_VIRTUAL */
- PciMsi::alloc_irq_vectors_affinity(dev, min_vecs, max_vecs, flags, NULL).

REQ-5: PciMsi::alloc_irq_vectors_affinity(dev, min, max, flags, affd) -> Result<i32, errno>:
- if flags & PCI_IRQ_AFFINITY: if !affd: affd = &irq_affinity{0}.
- else: if affd != NULL: WARN; affd = NULL.
- /* Try MSI-X first */
- if flags & PCI_IRQ_MSIX:
  - nvecs = PciMsi::enable_msix_range(dev, NULL, min, max, affd, flags).
  - if nvecs > 0: return Ok(nvecs).
- /* Then MSI */
- if flags & PCI_IRQ_MSI:
  - nvecs = PciMsi::enable_msi_range(dev, min, max, affd).
  - if nvecs > 0: return Ok(nvecs).
- /* INTx fallback (only if min_vecs == 1) */
- if flags & PCI_IRQ_INTX ∧ min == 1 ∧ dev.irq != 0:
  - if affd: irq_create_affinity_masks(1, affd).
  - pci_intx(dev, 1).
  - return Ok(1).
- return Err(nvecs).

REQ-6: PciMsi::enable_msi_range(dev, minvec, maxvec, affd) -> Result<i32, errno>:
- if !PciMsi::supported(dev, minvec) ∨ dev.current_state != PCI_D0: return -EINVAL.
- if dev.msix_enabled: WARN "can't enable MSI; MSI-X already on"; -EINVAL.
- if maxvec < minvec: -ERANGE.
- if dev.msi_enabled: WARN; -EINVAL.
- if !pci_msi_domain_supports(dev, 0, ALLOW_LEGACY): -ENOTSUPP.
- nvec = PciMsi::msi_vec_count(dev). if nvec < 0: return nvec. if nvec < minvec: -ENOSPC.
- pci_setup_msi_context(dev).
- if !pci_setup_msi_device_domain(dev, nvec): -ENODEV.
- if nvec > maxvec: nvec = maxvec.
- /* Retry-down loop on partial-success */
- loop:
  - if affd: nvec = irq_calc_affinity_vectors(minvec, nvec, affd). if nvec < minvec: -ENOSPC.
  - rc = PciMsi::capability_init(dev, nvec, affd).
  - if rc == 0: return Ok(nvec).
  - if rc < 0: return Err(rc).
  - if rc < minvec: -ENOSPC.
  - nvec = rc.

REQ-7: PciMsi::capability_init(dev, nvec, affd) -> i32:
- /* Reject multi-MSI on per-domain that doesn't support it */
- if nvec > 1 ∧ !pci_msi_domain_supports(dev, MSI_FLAG_MULTI_PCI_MSI, ALLOW_LEGACY): return 1.
- /* Hardware-disable MSI during setup */
- PciMsi::set_enable(dev, 0).
- masks = affd ? irq_create_affinity_masks(nvec, affd) : NULL.
- guard(msi_descs_lock)(&dev.dev).
- return PciMsi::capability_init_inner(dev, nvec, masks).

REQ-8: PciMsi::capability_init_inner(dev, nvec, masks) -> i32:
- ret = PciMsi::setup_msi_desc(dev, nvec, masks). if ret: return ret.
- entry = msi_first_desc(&dev.dev, MSI_DESC_ALL).
- pci_msi_mask(entry, msi_multi_mask(entry)) /* mask all before enable */.
- desc_copy = *entry /* for err path; domain alloc may free entry */.
- ret = pci_msi_setup_msi_irqs(dev, nvec, PCI_CAP_ID_MSI). if ret: goto err.
- ret = PciMsi::verify_entries(dev). if ret: goto err.
- dev.msi_enabled = 1.
- pci_intx_for_msi(dev, 0) /* disable INTx emulation */.
- PciMsi::set_enable(dev, 1) /* hardware-enable */.
- pcibios_free_irq(dev).
- dev.irq = entry.irq.
- return 0.
- err: pci_msi_unmask(&desc_copy, msi_multi_mask(&desc_copy)); pci_free_msi_irqs(dev).

REQ-9: PciMsi::setup_msi_desc(dev, nvec, masks) -> i32:
- desc = MsiDesc::zeroed.
- ctrl = config-read u16 at dev.msi_cap+PCI_MSI_FLAGS.
- /* Override broken devices */
- if dev.dev_flags & PCI_DEV_FLAGS_HAS_MSI_MASKING: ctrl |= PCI_MSI_FLAGS_MASKBIT.
- if pci_msi_domain_supports(dev, MSI_FLAG_NO_MASK, DENY_LEGACY): ctrl &= ~PCI_MSI_FLAGS_MASKBIT.
- desc.nvec_used = nvec.
- desc.pci.msi_attrib.is_64 = (ctrl & PCI_MSI_FLAGS_64BIT) != 0.
- desc.pci.msi_attrib.can_mask = (ctrl & PCI_MSI_FLAGS_MASKBIT) != 0.
- desc.pci.msi_attrib.default_irq = dev.irq.
- desc.pci.msi_attrib.multi_cap = FIELD_GET(PCI_MSI_FLAGS_QMASK, ctrl).
- desc.pci.msi_attrib.multiple = ilog2(roundup_pow_of_two(nvec)).
- desc.affinity = masks.
- desc.pci.mask_pos = dev.msi_cap + (is_64 ? PCI_MSI_MASK_64 : PCI_MSI_MASK_32).
- if desc.pci.msi_attrib.can_mask: read initial mask into desc.pci.msi_mask.
- return msi_insert_msi_desc(&dev.dev, &desc).

REQ-10: PciMsi::msix_capability_init(dev, entries, nvec, affd) -> i32:
- /* Enable MSI-X early + mask-all-vectors to prevent stray interrupts */
- PciMsi::msix_clear_and_set_ctrl(dev, 0, PCI_MSIX_FLAGS_MASKALL | PCI_MSIX_FLAGS_ENABLE).
- dev.msix_enabled = 1 /* setup may query */.
- ctrl = config-read u16 at msix_cap+PCI_MSIX_FLAGS.
- tsize = msix_table_size(ctrl) = (ctrl & PCI_MSIX_FLAGS_QSIZE) + 1.
- dev.msix_base = PciMsi::msix_map_region(dev, tsize).
- if !dev.msix_base: ret = -ENOMEM; goto out_disable.
- ret = PciMsi::msix_setup_interrupts(dev, entries, nvec, affd). if ret: goto out_unmap.
- pci_intx_for_msi(dev, 0).
- /* Mask all table entries (late) to handle broken Marvell NVMe quirk */
- if !pci_msi_domain_supports(dev, MSI_FLAG_NO_MASK, DENY_LEGACY): PciMsi::msix_mask_all(dev.msix_base, tsize).
- PciMsi::msix_clear_and_set_ctrl(dev, PCI_MSIX_FLAGS_MASKALL, 0) /* unmask all-vectors mask */.
- pcibios_free_irq(dev).
- return 0.
- out_unmap: iounmap(dev.msix_base).
- out_disable: dev.msix_enabled = 0; PciMsi::msix_clear_and_set_ctrl(dev, MASKALL | ENABLE, 0).

REQ-11: PciMsi::enable_msix_range(dev, entries, minvec, maxvec, affd, flags) -> i32:
- if maxvec < minvec: -ERANGE.
- if dev.msi_enabled: WARN "MSI already on"; -EINVAL.
- if dev.msix_enabled: WARN; -EINVAL.
- if !pci_msi_domain_supports(dev, MSI_FLAG_PCI_MSIX, ALLOW_LEGACY): -ENOTSUPP.
- if !PciMsi::supported(dev, nvec) ∨ dev.current_state != PCI_D0: -EINVAL.
- hwsize = PciMsi::msix_vec_count(dev). if hwsize < 0: return hwsize.
- if !pci_msix_validate_entries(dev, entries, nvec): -EINVAL /* duplicate / gap if MSIX_CONTIGUOUS required */.
- if hwsize < nvec:
  - if flags & PCI_IRQ_VIRTUAL: hwsize = nvec /* virtual-vector hack for some VF cases */.
  - else: nvec = hwsize.
- if nvec < minvec: -ENOSPC.
- pci_setup_msi_context(dev).
- if !pci_setup_msix_device_domain(dev, hwsize): -ENODEV.
- /* Retry-down loop */
- loop:
  - if affd: nvec = irq_calc_affinity_vectors(minvec, nvec, affd). if nvec < minvec: -ENOSPC.
  - rc = PciMsi::msix_capability_init(dev, entries, nvec, affd).
  - if rc == 0: return Ok(nvec).
  - if rc < 0: return Err(rc).
  - if rc < minvec: -ENOSPC.
  - nvec = rc.

REQ-12: PciMsi::msix_setup_msi_descs(dev, entries, nvec, masks) -> i32:
- vec_count = PciMsi::msix_vec_count(dev).
- desc = MsiDesc::zeroed.
- for i in 0..nvec:
  - desc.msi_index = entries ? entries[i].entry : i.
  - desc.affinity = masks ? &masks[i] : NULL.
  - desc.pci.msi_attrib.is_virtual = desc.msi_index >= vec_count.
  - PciMsi::prepare_msix_desc(dev, &desc).
  - if msi_insert_msi_desc(&dev.dev, &desc).is_err(): break.
- return ret.

REQ-13: PciMsi::prepare_msix_desc(dev, desc):
- desc.nvec_used = 1.
- desc.pci.msi_attrib.is_msix = 1.
- desc.pci.msi_attrib.is_64 = 1 /* MSI-X is always 64-bit */.
- desc.pci.msi_attrib.default_irq = dev.irq.
- desc.pci.mask_base = dev.msix_base.
- if !pci_msi_domain_supports(dev, MSI_FLAG_NO_MASK, DENY_LEGACY) ∧ !desc.pci.msi_attrib.is_virtual:
  - addr = pci_msix_desc_addr(desc) /* msix_base + msi_index * PCI_MSIX_ENTRY_SIZE=16 */.
  - desc.pci.msi_attrib.can_mask = 1.
  - if dev.dev_flags & PCI_DEV_FLAGS_MSIX_TOUCH_ENTRY_DATA_FIRST: writel(0, addr + PCI_MSIX_ENTRY_DATA) /* SUN NIU quirk */.
  - desc.pci.msix_ctrl = readl(addr + PCI_MSIX_ENTRY_VECTOR_CTRL).

REQ-14: PciMsi::msix_map_region(dev, n) -> Option<*Iomem>:
- table_offset = config-read u32 at msix_cap+PCI_MSIX_TABLE.
- bir = (table_offset & PCI_MSIX_TABLE_BIR) /* lower 3 bits */.
- flags = pci_resource_flags(dev, bir).
- if !flags ∨ (flags & IORESOURCE_UNSET): return None.
- table_offset &= PCI_MSIX_TABLE_OFFSET.
- phys = pci_resource_start(dev, bir) + table_offset.
- return ioremap(phys, n * PCI_MSIX_ENTRY_SIZE=16).

REQ-15: PciMsi::update_mask(desc, clear, set) /* MSI per-vector mask RMW */:
- dev = msi_desc_to_pci_dev(desc).
- if !desc.pci.msi_attrib.can_mask: return.
- raw_spin_lock_irqsave(&dev.msi_lock, flags).
- desc.pci.msi_mask &= ~clear.
- desc.pci.msi_mask |= set.
- config-write u32 at desc.pci.mask_pos = desc.pci.msi_mask.
- raw_spin_unlock_irqrestore.

REQ-16: irq_chip ops (pci_msi_template / pci_msix_template):
- chip:
  - name: "PCI-MSI" or "PCI-MSIX".
  - irq_startup: pci_irq_startup_msi / _msix (cond_startup_parent + unmask).
  - irq_shutdown: pci_irq_shutdown_msi / _msix (mask + cond_shutdown_parent).
  - irq_mask: pci_irq_mask_msi (calls pci_msi_mask(desc, BIT(irq - desc.irq))) / pci_irq_mask_msix (writel PCI_MSIX_ENTRY_CTRL_MASKBIT).
  - irq_unmask: pci_irq_unmask_msi / _msix.
  - irq_write_msi_msg: pci_msi_domain_write_msg (calls __pci_write_msi_msg).
  - flags: IRQCHIP_ONESHOT_SAFE.
- ops:
  - set_desc: pci_device_domain_set_desc (arg.desc = desc; arg.hwirq = desc.msi_index).
  - prepare_desc (MSI-X only): pci_msix_prepare_desc.
- info:
  - flags: MSI_FLAG_FREE_MSI_DESCS | MSI_FLAG_ACTIVATE_EARLY | MSI_FLAG_DEV_SYSFS | MSI_REACTIVATE | (MSI_FLAG_MULTI_PCI_MSI | MSI_FLAG_PCI_MSIX | MSI_FLAG_PCI_MSIX_ALLOC_DYN).
  - bus_token: DOMAIN_BUS_PCI_DEVICE_MSI / _MSIX.

REQ-17: PciMsi::shutdown_msi(dev):
- if !pci_msi_enable ∨ !dev.msi_enabled: return.
- PciMsi::set_enable(dev, 0) /* hardware-disable */.
- pci_intx_for_msi(dev, 1) /* re-enable INTx */.
- dev.msi_enabled = 0.
- /* Leave first desc unmasked (initial state) */
- desc = msi_first_desc(&dev.dev, MSI_DESC_ALL).
- if desc: pci_msi_unmask(desc, msi_multi_mask(desc)).
- dev.irq = desc.pci.msi_attrib.default_irq /* restore INTx pin */.
- pcibios_alloc_irq(dev).

REQ-18: PciMsi::shutdown_msix(dev):
- if !pci_msi_enable ∨ !dev.msix_enabled: return.
- if pci_dev_is_disconnected(dev): dev.msix_enabled = 0; return /* device gone; skip MMIO */.
- /* Mask all */
- for desc in msi_descs(dev.dev, MSI_DESC_ALL): pci_msix_mask(desc).
- PciMsi::msix_clear_and_set_ctrl(dev, PCI_MSIX_FLAGS_ENABLE, 0).
- pci_intx_for_msi(dev, 1).
- dev.msix_enabled = 0.
- pcibios_alloc_irq(dev).

REQ-19: PciMsi::free_msi_irqs(dev):
- pci_msi_teardown_msi_irqs(dev) /* releases all msi_descs + domain irqs */.
- if dev.msix_base: iounmap(dev.msix_base); dev.msix_base = NULL.

REQ-20: PciMsi::restore_msi_state(dev) / restore_msix_state — PM resume:
- /* MSI */
- if !dev.msi_enabled: return.
- entry = irq_get_msi_desc(dev.irq).
- pci_intx_for_msi(dev, 0); set_enable(dev, 0).
- if arch_restore_msi_irqs(dev): __pci_write_msi_msg(entry, &entry.msg).
- pci_msi_update_mask(entry, 0, 0).
- write back control word with QSIZE = entry.pci.msi_attrib.multiple + ENABLE.
- /* MSI-X */
- if !dev.msix_enabled: return.
- pci_intx_for_msi(dev, 0); msix_clear_and_set_ctrl(0, ENABLE | MASKALL).
- write_msg = arch_restore_msi_irqs(dev).
- for entry in msi_descs(dev.dev, MSI_DESC_ALL):
  - if write_msg: __pci_write_msi_msg(entry, &entry.msg).
  - pci_msix_write_vector_ctrl(entry, entry.pci.msix_ctrl).
- msix_clear_and_set_ctrl(MASKALL, 0).

REQ-21: PciMsi::irq_vector(dev, nr) -> i32:
- if !dev.msi_enabled ∧ !dev.msix_enabled: return nr == 0 ? dev.irq : -EINVAL.
- virq = msi_get_virq(&dev.dev, nr).
- return virq != 0 ? virq : -EINVAL.

REQ-22: PciMsi::msi_vec_count(dev) -> i32:
- if !dev.msi_cap: -EINVAL.
- msgctl = config-read u16 at msi_cap+PCI_MSI_FLAGS.
- return 1 << FIELD_GET(PCI_MSI_FLAGS_QMASK, msgctl) /* power-of-2, max 32 */.

REQ-23: PciMsi::msix_vec_count(dev) -> i32:
- if !dev.msix_cap: -EINVAL.
- ctrl = config-read u16 at msix_cap+PCI_MSIX_FLAGS.
- return (ctrl & PCI_MSIX_FLAGS_QSIZE) + 1 /* table-size up to 2048 */.

REQ-24: pci_msi_init / pci_msix_init (disable-on-probe, called from PciDev::init_capabilities):
- dev.msi_cap = pci_find_capability(dev, PCI_CAP_ID_MSI).
- if dev.msi_cap:
  - ctrl = config-read u16 at msi_cap+PCI_MSI_FLAGS.
  - if ctrl & PCI_MSI_FLAGS_ENABLE: config-write ctrl & ~ENABLE /* firmware-left-enabled */.
  - if !(ctrl & PCI_MSI_FLAGS_64BIT): dev.msi_addr_mask = DMA_BIT_MASK(32).
- dev.msix_cap = pci_find_capability(dev, PCI_CAP_ID_MSIX).
- if dev.msix_cap:
  - ctrl = config-read u16 at msix_cap+PCI_MSIX_FLAGS.
  - if ctrl & PCI_MSIX_FLAGS_ENABLE: config-write ctrl & ~ENABLE.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `msi_descs_lock_held` | INVARIANT | per-capability_init / msix_capability_init: msi_descs_lock(&dev.dev) held across desc-insert + domain-alloc. |
| `msi_lock_for_mask_rmw` | INVARIANT | per-pci_msi_update_mask: dev.msi_lock held during RMW of desc.pci.msi_mask + config write. |
| `msix_table_in_bar_range` | OOB | per-msix_map_region: phys = pci_resource_start(dev, bir) + table_offset; phys + n*16 ≤ pci_resource_end(dev, bir). |
| `msi_vec_count_power_of_two` | INVARIANT | per-pci_msi_vec_count: returns 1, 2, 4, 8, 16, or 32. |
| `msix_vec_count_bounded` | INVARIANT | per-pci_msix_vec_count: returns 1..=2048. |
| `enable_bit_cleared_during_setup` | INVARIANT | per-capability_init: PCI_MSI_FLAGS_ENABLE cleared before desc-setup, set after irq alloc succeeds. |
| `intx_emulation_disabled_under_msi` | INVARIANT | per-msi_enabled or msix_enabled: pci_intx(dev, 0) called; symmetric on shutdown. |
| `msix_base_iounmap_on_failure` | LEAK | per-msix_capability_init err path: out_unmap calls iounmap(dev.msix_base). |
| `restore_msi_state_idempotent` | INVARIANT | per-__pci_restore_msi_state: writes addr/data/ctrl from cached entry.msg + multiple. |

### Layer 2: TLA+

`models/pci/msi_alloc.tla`:
- States: { Free, MSI_Setup, MSI_Active, MSI_X_Setup, MSI_X_Active, INTx, Shutdown }.
- Transitions: alloc_irq_vectors / capability_init / msix_capability_init / shutdown.
- Properties:
  - `safety_no_msi_and_msix_simultaneous` — per-device: msi_enabled ∧ msix_enabled is unreachable.
  - `safety_enable_only_after_setup` — per-capability_init: ENABLE bit set ⟹ desc inserted ∧ domain irqs allocated.
  - `safety_shutdown_clears_enable` — per-shutdown: ENABLE bit cleared ∧ INTx restored.
  - `liveness_alloc_terminates` — per-pci_alloc_irq_vectors: retry-down loop converges in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `PciMsi::alloc_irq_vectors` post: Ok(n) ⟹ n ≥ min_vecs ∧ n ≤ max_vecs ∧ dev.msi_enabled ∨ dev.msix_enabled ∨ (n == 1 ∧ INTx) | `PciMsi::alloc_irq_vectors` |
| `PciMsi::capability_init` post: rc == 0 ⟹ dev.msi_enabled ∧ PCI_MSI_FLAGS_ENABLE set ∧ msi_descs inserted | `PciMsi::capability_init` |
| `PciMsi::msix_capability_init` post: rc == 0 ⟹ dev.msix_enabled ∧ dev.msix_base != NULL ∧ MASKALL cleared | `PciMsi::msix_capability_init` |
| `PciMsi::setup_msi_desc` post: returned desc has is_64, can_mask, multi_cap, multiple, mask_pos consistent with PCI_MSI_FLAGS contents | `PciMsi::setup_msi_desc` |
| `PciMsi::prepare_msix_desc` post: desc.pci.msi_attrib.is_msix = 1 ∧ is_64 = 1 ∧ mask_base = dev.msix_base | `PciMsi::prepare_msix_desc` |
| `PciMsi::shutdown_msi` post: dev.msi_enabled = false ∧ INTx re-enabled ∧ dev.irq = default_irq | `PciMsi::shutdown_msi` |
| `PciMsi::shutdown_msix` post: dev.msix_enabled = false ∧ all desc masked ∧ INTx re-enabled | `PciMsi::shutdown_msix` |
| `PciMsi::free_msi_irqs` post: dev.msix_base == NULL ∧ msi_descs freed | `PciMsi::free_msi_irqs` |
| `PciMsi::verify_entries` post: ∀ entry. (address_hi << 32 | address_lo) & ~dev.msi_addr_mask == 0 | `PciMsi::verify_entries` |

### Layer 4: Verus/Creusot functional

`PciMsi::alloc_irq_vectors(dev, min_vecs, max_vecs, flags)` ↔ upstream `pci_alloc_irq_vectors`: for any PCI device with msi_cap and/or msix_cap discovered, the number of allocated vectors, the assigned Linux IRQs, the MsiMsg payloads (address_lo/hi, data), and the resulting MSI/MSI-X capability-register state are byte-identical with upstream. Encoded as Verus refinement: `forall dev cfg. rookery_alloc(dev, cfg) == upstream_alloc(dev, cfg)`.

### hardening

(Inherits row-1 features from `drivers/pci/00-overview.md` § Hardening.)

MSI/MSI-X reinforcement:

- **MSI / MSI-X hardware-disabled during setup** — PCI_MSI_FLAGS_ENABLE cleared before `setup_msi_desc`; PCI_MSIX_FLAGS_MASKALL set before MSI-X table programmed (defense against per-stray-interrupt during setup).
- **MSI-X table mask-all before unmask-individual** — late `msix_mask_all` writes PCI_MSIX_ENTRY_CTRL_MASKBIT to every table entry's VECTOR_CTRL before the all-vectors MASKALL is cleared (defense against per-Marvell-NVMe-style quirk where table mask bits matter even when MSI-X is disabled).
- **Per-vector mask under msi_lock** — `pci_msi_update_mask` holds `dev.msi_lock` (raw_spinlock_t) across desc.pci.msi_mask RMW + config-space write (defense against per-concurrent mask flicker on multi-vector multi-CPU).
- **msi_descs_lock across setup** — `guard(msi_descs_lock)(&dev.dev)` held across desc-insertion + domain-alloc (defense against per-concurrent allocator on same device).
- **MSI / MSI-X mutual exclusion** — `enable_msi_range` rejects if `dev.msix_enabled`; `enable_msix_range` rejects if `dev.msi_enabled` (defense against per-driver-bug double-enable).
- **PCI_BUS_FLAGS_NO_MSI walked up-tree** — `pci_msi_supported` walks `dev.bus → bus.parent → …` checking NO_MSI flag (defense against per-bridge-that-cannot-forward-MSI silently routing MSI writes upstream).
- **dev.msi_addr_mask verified post-arch-assign** — `msi_verify_entries` checks every entry's address ≤ `dev.msi_addr_mask`; rejects with -EIO (defense against per-32-bit-MSI device receiving 64-bit message address).
- **INTx emulation disabled while MSI active** — `pci_intx_for_msi(dev, 0)` on enable; `pci_intx_for_msi(dev, 1)` on shutdown (defense against per-double-delivery via INTx wire + MSI message).
- **Retry-down on partial-success bounded** — `__pci_enable_msi_range` / `_msix_range` loop terminates when rc < minvec (returns -ENOSPC) or rc == 0 (success) (defense against per-infinite-retry on broken irq_domain).
- **Multi-MSI rejected on non-supporting domain** — `nvec > 1 ∧ !MSI_FLAG_MULTI_PCI_MSI` returns 1 (not -EINVAL) so caller retries with nvec=1 (defense against per-x86-default-domain not supporting multi-MSI).
- **MSI / MSI-X ENABLE bits cleared on probe** — `pci_msi_init` / `pci_msix_init` clear ENABLE bits at device-add time (defense against per-firmware-left-enabled MSI/MSI-X masking driver-init interrupt handling).
- **Disconnected-device guard in shutdown_msix** — `pci_dev_is_disconnected(dev)` short-circuits before MMIO writes to the MSI-X table (defense against per-PCI-error-recovery / hot-remove MMIO crash).
- **iounmap on failure** — `out_unmap` path in `msix_capability_init` releases `dev.msix_base` ioremap (defense against per-VA-leak on capability_init failure).
- **DOMAIN_BUS_PCI_DEVICE_MSI vs _MSIX bus tokens distinct** — per-domain template tags distinguish MSI / MSI-X so x86/IOMMU code can route MSI differently from MSI-X (defense against per-misrouted-message in IOMMU-remap).

