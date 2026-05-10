# Tier-2: drivers/mtd — Memory Technology Devices (NOR + NAND + SPI-NOR + SPI-NAND + UBI + JFFS2 + ubifs + chardev + mtdblock)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/mtd/
  - include/linux/mtd/
  - include/uapi/mtd/
-->

## Summary

Tier-2 wrapper for MTD — flash-memory abstraction underneath every BIOS/UEFI SPI-NOR boot ROM, every embedded NAND, every SoC raw-NAND controller, every SD-flashed bootloader image. Components: **core** (`mtdcore.c`, `mtdsuper.c`, `mtdpart.c`, `mtdconcat.c`, `mtdchar.c` (`/dev/mtd<N>` chardev), `mtdblock.c` (`/dev/mtdblock<N>` block emulation), `mtdoops.c` (oops-on-flash for crash dumps), `mtdswap.c` (swap on MTD), `cmdlinepart.c`, `ofpart_*.c`, `redboot.c`, `parsers/`), **chips** (`chips/`: NOR-flash protocol drivers — CFI / Intel / AMD), **maps** (`maps/`: per-bus mapping drivers — `physmap.c` for memory-mapped NOR, `intel-vr-nor.c`, `lantiq-flash.c`, `pcmciamtd.c`, `sa1100-flash.c`, etc.), **devices** (`devices/`: `block2mtd.c` (use a block device as MTD), `mtdram.c` (RAM-backed MTD for testing), `phram.c`, `slram.c`), **NAND** (`nand/`: `raw/` — raw NAND flash bus ops + per-controller drivers (~50, mostly ARM), `onenand/` — Samsung OneNAND, `spi/` — SPI-NAND), **SPI-NOR** (`spi-nor/`: SPI-NOR flash chips — Micron, Macronix, Winbond, Spansion, ISSI, GigaDevice, etc., consumed via `drivers/spi/spi-mem.md`), **UBI** (`ubi/`: UBI Unsorted Block Images — wear-leveling + bad-block management on top of raw NAND; provides volume layer underneath UBIFS), **NVMEM** (cross-ref `drivers/nvmem/`).

Filesystems on MTD (separate Tier-2): JFFS2 (`fs/jffs2/`), UBIFS (`fs/ubifs/`), squashfs-on-MTD, cramfs-on-MTD.

## Compatibility contract — outline

- `/dev/mtd<N>` chardev IOCTLs byte-identical: `MEMGETINFO`, `MEMERASE`, `MEMERASE64`, `MEMGETREGIONCOUNT`, `MEMGETREGIONINFO`, `MEMLOCK`, `MEMUNLOCK`, `MEMISLOCKED`, `MEMGETBADBLOCK`, `MEMSETBADBLOCK`, `MEMWRITEOOB`, `MEMREADOOB`, `MEMWRITE`, `MEMREAD`, `MEMGETOOBSEL`, `MEMGETOOBLAYOUT`, `MTDFILEMODE`, `OTPSELECT`, `OTPGETREGIONCOUNT`, `OTPGETREGIONINFO`, `OTPLOCK`, `MEMERASE_USER`, `MEMWRITEOOB64`, `MEMREADOOB64` (mtd-utils + flashcp + flash_eraseall + nandwrite + nanddump consume).
- `/dev/mtdblock<N>` block device for read/write at sector granularity.
- `/sys/class/mtd/mtd<N>/{name, type, size, erasesize, writesize, oobsize, numeraseregions, flags, dev, offset, ...}` byte-identical.
- UBI `/dev/ubi<N>_<vol>` chardev + `UBI_*` IOCTLs byte-identical (ubi-utils consumes).
- DT `partitions` binding + cmdline `mtdparts=` parsed identically.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/mtd/core.md` | `mtdcore.c` + `mtdpart.c` + `mtdconcat.c`: framework |
| `drivers/mtd/chardev-block.md` | `mtdchar.c` + `mtdblock.c`: `/dev/mtd*` + `/dev/mtdblock*` |
| `drivers/mtd/parsers.md` | `cmdlinepart.c` + `ofpart_*.c` + `parsers/`: partition parsers |
| `drivers/mtd/oops-swap.md` | `mtdoops.c` + `mtdswap.c` |
| `drivers/mtd/maps.md` | `maps/`: per-bus mapping (physmap NOR, etc.) |
| `drivers/mtd/devices.md` | `devices/`: block2mtd + mtdram + phram |
| `drivers/mtd/spi-nor.md` | `spi-nor/`: SPI-NOR flash chip support |
| `drivers/mtd/nand-raw.md` | `nand/raw/`: raw NAND flash bus ops + per-controller |
| `drivers/mtd/nand-onenand.md` | `nand/onenand/`: Samsung OneNAND |
| `drivers/mtd/nand-spi.md` | `nand/spi/`: SPI-NAND |
| `drivers/mtd/ubi.md` | `ubi/`: UBI volume layer |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: `/dev/mtd*`, `/dev/mtdblock*`, `/dev/ubi*` UAPI byte-identical.
- REQ-O2: SPI-NOR chip-id table preserves Micron/Macronix/Winbond/Spansion/ISSI/GigaDevice/SST/Atmel coverage.
- REQ-O3: UBI volume layer source-compat for ubifs.
- REQ-O4: TLA+ models (per-MTD erase + write atomicity; UBI bad-block management + wear-leveling convergence; UBI volume update atomicity).
- REQ-O5: AC: BIOS SPI-NOR enumerates as `/dev/mtd<N>`; flashrom dump-and-restore round-trips bit-identical.
- Hardening: `/dev/mtd*` write requires CAP_SYS_RAWIO + LSM mediation (defends against re-flashing BIOS/firmware); UBI volume create requires CAP_SYS_ADMIN.

## Out of Scope
Implementation code; most ARM SoC NAND controllers; 32-bit-only paths.
