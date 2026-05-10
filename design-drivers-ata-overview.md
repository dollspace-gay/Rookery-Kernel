---
title: "Tier-2: drivers/ata — libata + AHCI + per-controller LLDs (sata + pata)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for libata + AHCI — every SATA SSD/HDD/optical, every legacy PATA/IDE drive (kept for legacy compat), every eSATA dock. Components: **libata core** (`libata-core.c`, `libata-eh.c`, `libata-scsi.c`, `libata-trace.c`, `libata-pmp.c`, `libata-acpi.c`, `libata-zpodd.c`, `libata-sata.c`, `libata-sff.c`, `libata-pata-timings.c`, `libata-transport.c`: ATA command + EH + SCSI bridging + Port Multiplier + ACPI + Zero-Power Optical), **AHCI** (`ahci.c` + `ahci.h` + `ahci_platform.c` + `ahci_dwc.c` + per-vendor AHCI variants `acard-ahci.c`, `ahci_brcm.c` (out of v0 ARM), `ahci_ceva.c` (out of v0), `ahci_imx.c` (out of v0), `ahci_mtk.c`, `ahci_mvebu.c`, `ahci_qoriq.c`, `ahci_st.c`, `ahci_sunxi.c`, `ahci_tegra.c`, `ahci_xgene.c`, `sata_highbank.c` — most ARM compile-gated off), **per-controller LLDs** (~50, v0 set: `sata_inic162x.c`, `sata_mv.c`, `sata_nv.c`, `sata_promise.c`, `sata_sil.c`, `sata_sil24.c`, `sata_sis.c`, `sata_svw.c`, `sata_sx4.c`, `sata_uli.c`, `sata_via.c`, `sata_vsc.c`, plus PATA: `pata_*` ~30 — most legacy ISA/VLB/PCI compile-gated off; v0 keep: `pata_jmicron.c`, `pata_marvell.c`, `pata_nvidia.c`, `pata_via.c`, `pata_amd.c`, `pata_atiixp.c`, `pata_oldpiix.c`).

### compatibility contract — outline

- ATA disk presents as `/dev/sd<X>` via SCSI-bridge (libata-scsi). Naming + minor-numbering preserved.
- AHCI controller binds via `pci_driver` w/ class 0x010601 (SATA-AHCI).
- `/sys/class/ata_port/ata<N>/{ahci_port_cmd, ahci_host_caps, idle_irq, link_pm_policy, em_*, sw_*, ...}` byte-identical (smartctl + sosreport consume).
- `/sys/class/ata_link/link<N>/{hw_sata_spd_limit, sata_spd_limit, sata_spd, ...}` byte-identical.
- `/sys/class/ata_device/dev<N>/{class, id, gscr, ering, ...}` byte-identical.
- DPM `/sys/class/scsi_host/host<N>/link_power_management_policy` byte-identical.
- Cmdline `ahci.mobile_lpm_policy=`, `ahci.ignore_sss=` parsed identically.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/ata/libata-core.md` | `libata-core.c` + `libata-eh.c`: ATA command + EH |
| `drivers/ata/libata-scsi.md` | `libata-scsi.c`: SCSI translation |
| `drivers/ata/libata-pmp.md` | `libata-pmp.c`: SATA Port Multiplier |
| `drivers/ata/libata-acpi.md` | `libata-acpi.c`: ACPI integration |
| `drivers/ata/libata-zpodd.md` | `libata-zpodd.c`: Zero-Power Optical |
| `drivers/ata/libata-sff-sata.md` | `libata-sff.c` + `libata-sata.c`: SFF + SATA helpers |
| `drivers/ata/libata-pata-timings.md` | `libata-pata-timings.c` |
| `drivers/ata/libata-transport.md` | `libata-transport.c` |
| `drivers/ata/ahci-core.md` | `ahci.c` + `ahci.h`: standard AHCI |
| `drivers/ata/ahci-platform.md` | `ahci_platform.c`: platform-AHCI shim |
| `drivers/ata/sata-vendor-lld.md` | `sata_*.c`: per-vendor SATA LLDs |
| `drivers/ata/pata-vendor-lld.md` | `pata_*.c`: PATA LLDs (legacy) |

### compatibility outline / ac / verification / hardening

- REQ-O1: SATA + PATA disks enumerate as `/dev/sd<X>` via libata-scsi bridge (existing tools work).
- REQ-O2: AHCI binds + every supported feature (NCQ, LPM, hot-plug, port-multiplier, SED-Opal pass-through) works.
- REQ-O3: TLA+ models (libata EH escalation: ATA-link-reset → port-reset → host-reset; concurrent IO + EH safe; NCQ tag completion ordering).
- REQ-O4: AC: SATA SSD enumerates + smartctl shows correct identify + IO works at NCQ-saturated throughput.
- Hardening: row-1; ATA passthrough via SCSI SG_IO write-cmds gated CAP_SYS_RAWIO; SED-Opal mediation (cross-ref `block/00-overview.md`).

