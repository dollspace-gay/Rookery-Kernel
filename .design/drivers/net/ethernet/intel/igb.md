# Tier-3: drivers/net/ethernet/intel/igb/{igb_main,igb_ethtool,igb_ptp,igb_hwmon,igb_xsk,e1000_82575,e1000_i210,e1000_mac,e1000_phy,e1000_nvm,e1000_mbx}.c — Intel 82575/82576/82580/I210/I211/I340/I350/I354 1 Gbps Ethernet (every enterprise IPMI/BMC interface, every server embedded NIC, every industrial/embedded controller)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
upstream-paths:
  - drivers/net/ethernet/intel/igb/igb_main.c                  (~8500 lines: netdev ops, NAPI, RX/TX, IRQ, SR-IOV via mailbox)
  - drivers/net/ethernet/intel/igb/igb_ethtool.c               (~3300 lines: ethtool ops)
  - drivers/net/ethernet/intel/igb/igb_ptp.c                   (~1200 lines: PHC + per-packet timestamping)
  - drivers/net/ethernet/intel/igb/igb_hwmon.c                 (~230 lines: temp sensor)
  - drivers/net/ethernet/intel/igb/igb_xsk.c                   (AF_XDP zerocopy)
  - drivers/net/ethernet/intel/igb/e1000_82575.c               (~3000 lines: 82575/82576/82580 silicon family)
  - drivers/net/ethernet/intel/igb/e1000_i210.c                (~910 lines: I210/I211 silicon family)
  - drivers/net/ethernet/intel/igb/e1000_mac.c                 (~2000 lines: cross-silicon MAC ops)
  - drivers/net/ethernet/intel/igb/e1000_phy.c                 (~3000 lines: PHY ops including 82580/I35x integrated PHY)
  - drivers/net/ethernet/intel/igb/e1000_nvm.c                 (NVRAM)
  - drivers/net/ethernet/intel/igb/e1000_mbx.c                 (PF↔VF mailbox for SR-IOV — sibling igbvf VF driver)
  - drivers/net/ethernet/intel/igb/igb.h
-->

## Summary

igb is the Intel 1 Gigabit Ethernet driver covering server-class silicon: 82575/82576/82580 (first-gen, 2007-2010), I210/I211 (single-port server LOM, 2013+), I340/I350 (4-port server, 2011+ — workhorse of the era), I354 (Atom-SoC integrated). Every enterprise server motherboard since the late 2000s has at least one igb-driven port — typically the IPMI/BMC management interface (1Gb dedicated LAN for iLO/iDRAC/IPMI), the host-side OS management network, or embedded SoC LAN. Still ships in current enterprise gear. Also widely deployed in industrial controllers (because of strong PTP support), Cisco UCS service-processor mgmt, and high-end NAS appliances.

igb is the SR-IOV-capable sibling of the older `e1000e` (no-SR-IOV) and the older `igb`-era 82575+ family is where Intel first introduced PF/VF mailbox-based SR-IOV for the consumer/small-server market (long before XL710/E810).

Source: ~25500 lines / 11 .c files.

This Tier-3 covers all of igb.

## Rust translation posture

igb is structurally similar to ixgbe (mailbox-register SR-IOV pattern, per-silicon HW ops) but smaller. Translation:

- **`igb-core` crate** — netdev, NAPI, RX/TX, SR-IOV PF, ethtool, PTP, AF_XDP.
- **Per-silicon `IgbSilicon` trait** — 82575/82576/82580/I210/I211/I340/I350/I354 each implementing MAC + PHY + NVM ops.
- **`IgbAdapter<Phase>` typed lifecycle.**
- **PF↔VF mailbox typed** like ixgbe.
- **PTP-grade timestamping** typed `Phc` (similar to igc but older silicon — sub-µs precision).
- **AF_XDP zerocopy** standard pattern (newer addition).

Grsec is mandatory. igb has CVE history (older silicon attack surface; multiple mailbox + ethtool bugs over the years).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct igb_adapter` | per-netdev priv | `IgbAdapter<Phase>` |
| `struct e1000_hw` | HW abstraction (shared with e1000e) | `E1000Hw<S>` |
| `struct igb_ring` | per-direction ring | `Ring<Direction>` |
| `struct igb_q_vector` | per-MSI-X NAPI | `QVector` |
| `igb_probe(pdev, ent)` / `_remove(pdev)` | PCI probe/remove | `IgbAdapter::probe` / `Drop` |
| `igb_open(netdev)` / `_close(netdev)` | open/close | `IgbAdapter::open` / `close` |
| `igb_xmit_frame_ring(skb, tx_ring)` | TX | `Ring<Tx>::xmit` |
| `igb_poll(napi, budget)` | NAPI | `Napi::poll` |
| `igb_msix_other(irq, data)` / `_msix_ring(irq, data)` | MSI-X | `Irq::*` |
| `igb_set_default_mac_filter(adapter)` / `_add_mac_filter(...)` | MAC filter | `Filter::*` |
| `igb_setup_rx_mode(netdev)` | rx mode | `Filter::set_rx_mode` |
| `igb_init_hw_82575(...)` / `_init_hw_i210(...)` etc. | per-silicon init | `Silicon::init_hw` |
| `igb_setup_link_*` / `_check_for_link_*` | link mgmt | `Link::*` |
| `igb_pci_sriov_configure(pdev, num_vfs)` / `_enable_sriov(...)` / `_disable_sriov(...)` | SR-IOV | `Sriov::*` |
| `igb_msg_task(adapter)` / `_rcv_msg_from_vf(...)` / `_set_vf_*` | VF mailbox handler | `VfMbx::*` |
| `igb_ptp_init(adapter)` / `_release(adapter)` / `_tx_hwtstamp_work(work)` | PTP | `Ptp::*` |
| `igb_xdp(netdev, xdp)` / `_xdp_xmit(...)` | XDP | `Xdp::*` |
| `igb_xsk_pool_setup(adapter, pool, qid)` / `_xsk_wakeup(...)` | AF_XDP | `Xsk::*` |
| `igb_get_drvinfo(...)` / `_get_ringparam(...)` / `_get_coalesce(...)` | ethtool | `Ethtool::*` |
| `igb_hwmon_init(adapter)` | hwmon | `Hwmon::*` |
| `igb_nvm_*` | NVRAM | `Nvm::*` |
| `igb_io_error_detected(...)` / `_io_slot_reset(...)` / `_io_resume(...)` | AER handlers | `Aer::*` |
| `e1000_mbx_*` | mailbox primitives | `Mbx::*` |
| `e1000_phy_*` | PHY ops | `Phy::*` |

## Compatibility contract

REQ-1: PCI ID table — every 82575/82576/82580/I210/I211/I340/I350/I354 silicon. Frozen.

REQ-2: Per-silicon HW ops frozen against silicon datasheets.

REQ-3: SR-IOV (where supported — 82576/82580/I350) — up to 8 VFs per port via mailbox; sibling `igbvf` VF driver runs in guests.

REQ-4: PF↔VF mailbox — opcodes per `e1000_mbx.h`; VF_RESET, VF_GET_MAC, VF_SET_MAC, VF_GET_VLAN, VF_SET_VLAN, VF_SET_MULTICAST.

REQ-5: PHC strong timestamping (newer silicon — I210/I350) — sub-µs sync precision.

REQ-6: AF_XDP zerocopy (newer addition).

REQ-7: XDP_DRV native.

REQ-8: Wake-on-LAN — magic packet + unicast filter.

REQ-9: AER — pci_error_handlers.

REQ-10: I210/I211 with on-die flash NVRAM (no external EEPROM) — different NVM path than 82575/82576 with external EEPROM.

REQ-11: Integrated PHY on 82580/I340/I350 (newer silicon doesn't need external PHY).

REQ-12: ethtool full surface (link, RSS, coalesce, ring, NVRAM, stats).

## Acceptance Criteria

- [ ] AC-1: I350-T4 (4-port 1G) probes; all 4 ports up; iperf3 saturates 1G per port.
- [ ] AC-2: SR-IOV: 7 VFs on I350; igbvf in guest works.
- [ ] AC-3: PTP: ptp4l syncs to NIC PHC.
- [ ] AC-4: XDP_DRV smoke.
- [ ] AC-5: AF_XDP zerocopy.
- [ ] AC-6: WOL: ethtool -s eth0 wol g; suspend; magic packet wakes.
- [ ] AC-7: AER injection recovery.
- [ ] AC-8: BMC scenario: igb-driven IPMI port on a Dell PowerEdge; BMC web UI reachable; OS-side traffic isolated.

## Architecture

**Standard Intel netdev pattern.** Per-CPU NAPI + per-CPU RX/TX rings + MSI-X. Smaller queue count (4-8) vs enterprise NICs. Page-pool RX.

**Per-silicon HW ops.** `e1000_hw.mac.ops.X()` dispatch — same pattern as ixgbe. 82575 family + I210 family + I35x family each have distinct init paths + workarounds.

**SR-IOV (82576/82580/I350).** PF reserves resources for VFs at SR-IOV enable; per-VF MAC/VLAN/multicast via mailbox. `igbvf` runs in guests.

**PF↔VF mailbox.** Doorbell-driven; smaller opcode set than ixgbe (fewer features).

**PHC (newer silicon).** Sub-µs precision PTP on I210/I350+. Per-packet TX/RX timestamps in dedicated regs.

**AF_XDP / XDP_DRV.** Standard pattern at NAPI poll path.

**Hwmon.** I210 has on-die temp sensor exposed via hwmon class.

## Hardening

- Per-silicon init applies right HW-ops set.
- PF↔VF mailbox payload validated against schema.
- Mailbox opcode allowlist.
- AER recovery rate-limited.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — ethtool, sysfs ingress bounded.
- **PAX_KERNEXEC** — netdev_ops, ethtool_ops, per-silicon op tables, mbx dispatch, PTP ops, XDP ops, AER handlers all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — ethtool / netlink / sysfs entry paths per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `igb_adapter` refcount, per-q_vector refcount, per-ring refcount, per-VF refcount saturating.
- **PAX_MEMORY_SANITIZE** — VF-context cleared on disable_sriov. PHC sample buffer cleared.
- **PAX_UDEREF** — SMAP/SMEP on every user-facing entry.
- **PAX_RAP / kCFI** — netdev_ops, ethtool_ops, per-silicon op tables, mbx handlers, PTP ops, AER handlers all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs scrubbed.
- **GRKERNSEC_DMESG** — verbose logs (VF state changes, PHC samples) restricted from non-root.
- **CAP_NET_ADMIN required** — every state-changing op.
- **PF→VF mailbox per-opcode allowlist + size strict** — closes "compromised VF tricks PF" class (same as ixgbe).
- **VF spoof-check HW-enforced** — same pattern.
- **PHC CAP_SYS_TIME** — clock adjust requires CAP_SYS_TIME.
- **NVRAM access CAP_SYS_ADMIN + LOCKDOWN** — closes MAC-change-via-NVRAM-write attack.
- **AER recovery rate-limit** — 3 in 60s.
- **WOL config CAP_NET_ADMIN** — closes "WOL-prevents-shutdown" class.
- **PHY register write allowlist** — direct PHY register writes restricted to documented allowlist.
- **BMC isolation respected** — when igb port is shared with BMC (NCSI side-band), driver respects BMC ownership and refuses to modify configs the BMC has claimed (closes "host OS reconfigures BMC's port" attack).
- **NCSI passthru opcode allowlist** — when NCSI is active, frame types allowed to traverse the side-band are restricted; closes a path where host OS could inject hostile traffic into BMC's network.
- **VF VLAN filter cap strict** — VF cannot install more VLAN filters than its quota.

Per-doc rationale: igb is the IPMI/BMC interface NIC on every enterprise server — this gives it a unique trust posture. The igb port is often shared between host OS and BMC (via NCSI side-band); BMC traffic must be isolated from host OS attempts to read/modify. The Rust translation's per-silicon trait + typed PF↔VF mailbox + NCSI passthru opcode allowlist + BMC-ownership respect close the dominant CVE classes structurally. Lower-priority than enterprise PFs because feature stack is smaller, but the BMC/IPMI use case makes correctness essential (a compromised BMC interface is a remote-server-takeover).

## Open Questions

- [ ] Q1: 82575/82576 (first-gen, 2007-2010) is mostly retired. Maintain or drop? Recommendation: maintain — still in long-life embedded.
- [ ] Q2: NCSI (Network Controller Sideband Interface) for BMC sharing — needs explicit design for NCSI passthru filter rules.
- [ ] Q3: igbvf (sibling VF) — separate Tier-3.

## Verification

- **Kani SAFETY**: prove silicon-trait dispatch exhaustive; PF→VF mailbox payload validation per-opcode.
- **TLA+**: model NCSI passthru rules + concurrent host-OS reconfig; check BMC isolation invariant.
- **Verus**: functional spec of PF→VF mailbox message validation; for valid msg, dispatches; for malformed, returns -EINVAL.
- **Kani+Verus**: invariant that every active VF has tracked per-VF state; cleanup at disable_sriov removes all.
- **Integration**: I350 4-port + SR-IOV stress + PTP sync + NCSI/BMC isolation test on a real iDRAC-equipped server.
- **Fuzz**: PF→VF mailbox fuzzer; NCSI filter fuzzer.
- **Penetration**: from host OS, attempt to disrupt BMC traffic via NCSI passthru — refused.

## Out of Scope

- igbvf (VF driver) — separate Tier-3
- e1000e (Intel 1G older, no-SR-IOV) — separate Tier-3
- ixgbe / i40e / ice / igc — covered in their own docs
- NCSI framework details — separate net/ncsi/ Tier-3
- BMC/IPMI internals — out of scope
