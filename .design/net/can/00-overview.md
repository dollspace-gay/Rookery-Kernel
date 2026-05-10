# Tier-2: net/can — CAN bus stack (raw + bcm + isotp + j1939 + gw + per-driver controllers)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/can/
  - drivers/net/can/
  - include/linux/can/
  - include/uapi/linux/can/
-->

## Summary

Tier-2 wrapper for CAN (Controller Area Network) — automotive + industrial bus. AF_CAN socket family with 4 protocols: **CAN_RAW** (`raw.c` — raw CAN frames), **CAN_BCM** (`bcm.c` — Broadcast Manager: cyclic + filter rules), **CAN_ISOTP** (`isotp.c` — ISO-TP / ISO-15765-2: segmented multi-frame transport up to 4GB used by UDS automotive diagnostics), **CAN_J1939** (`j1939/` — SAE J1939 / NMEA 2000 truck/marine higher-layer protocol with 8-bit priority + 18-bit PGN). Plus **CAN-GW** (`gw.c` — netlink-controlled CAN-frame gateway between CAN interfaces with re-write rules). Plus per-driver controllers under `drivers/net/can/` (m_can, ifi_canfd, peak_canfd, kvaser_pciefd, mcp251x, mcp251xfd, sja1000/, c_can, flexcan, etc. — ~30 controllers + USB CAN dongles).

## Compatibility contract — outline

- AF_CAN socket protocols `CAN_RAW`, `_BCM`, `_TP16`, `_TP20`, `_MCNET`, `_ISOTP`, `_J1939` byte-identical.
- `struct can_frame`, `canfd_frame`, `canxl_frame`, `sockaddr_can`, `can_filter` UAPI byte-identical.
- CAN_BCM ops `TX_SETUP`, `TX_DELETE`, `TX_READ`, `TX_SEND`, `RX_SETUP`, `RX_DELETE`, `RX_READ` byte-identical.
- ISO-TP setsockopt + sockaddr_can_isotp byte-identical (used by UDS / OBD-II / KWP2000 diagnostic tools).
- J1939 `SO_J1939_*` setsockopt byte-identical.
- `/proc/net/can/{rcvlist_*,reset_stats,stats,version}` byte-identical.
- CAN-GW netlink controlled via `cangw` userspace tool; netlink message format byte-identical.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `net/can/af-can.md` | `af_can.c`: AF_CAN root |
| `net/can/raw.md` | `raw.c`: CAN_RAW |
| `net/can/bcm.md` | `bcm.c`: Broadcast Manager |
| `net/can/isotp.md` | `isotp.c`: ISO-TP segmented transport |
| `net/can/j1939.md` | `j1939/`: SAE J1939 |
| `net/can/gw.md` | `gw.c`: CAN gateway |
| `net/can/proc.md` | `proc.c` + `gw.c` netlink |
| `drivers/net/can/m_can.md` | `drivers/net/can/m_can/`: Bosch M_CAN IP controller (CAN-FD) |
| `drivers/net/can/peak.md` | `drivers/net/can/peak_*.c`: PEAK PCI/USB CAN |
| `drivers/net/can/kvaser.md` | `drivers/net/can/kvaser_*.c`: Kvaser PCIe CAN |
| `drivers/net/can/mcp25xxfd.md` | `drivers/net/can/mcp251xfd/`: Microchip MCP2517FD/MCP2518FD SPI |
| `drivers/net/can/usb.md` | `drivers/net/can/usb/`: USB CAN dongles (PEAK, GS_USB, etcan, esd_usb2, ucan) |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: AF_CAN UAPI byte-identical (can-utils + python-can + cangw + isotp-userspace + j1939-utils consume unchanged).
- REQ-O2: CAN-FD + CAN-XL data-frame variants supported on supporting controllers.
- REQ-O3: BCM cyclic + filter behavior identical.
- REQ-O4: ISO-TP segmentation + reassembly + flow-control identical.
- REQ-O5: TLA+ models (BCM cyclic-tx scheduling, ISO-TP segmentation-reassembly state, J1939 transport state machine).
- REQ-O6: AC: `cangen vcan0` + `candump vcan0` round-trip; UDS diagnostic tool (`isotp-tools`) over real OBD-II adapter works.
- Hardening: row-1 features; per-namespace CAN scoping; `cangw` requires CAP_NET_ADMIN; CAN_RAW with CAN_RAW_FD_FRAMES default-on requires CAP_NET_RAW only when bound to physical interface.

## Out of Scope
- Implementation code; ARM-only SoC CAN drivers (most compile-gated off for v0); 32-bit-only paths
