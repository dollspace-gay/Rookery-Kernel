# Tier-3: drivers/net/wireless/ath/ath11k/{core,pci,ahb,mhi,hal,htc,wmi,dp,ce,dbring,mac,reg,peer,spectral,thermal,trace,debugfs,coredump,fw,qmi}.c — Qualcomm Atheros ath11k WiFi 6/6E (IPQ8074/QCA6390/WCN6750/QCN9074/IPQ6018 — TP-Link/Linksys/Synology AP routers, Pixel 5+ phones, ChromeOS, Snapdragon-X laptops)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/wireless/00-overview.md
sibling: drivers/net/wireless/intel/iwlwifi.md
upstream-paths:
  - drivers/net/wireless/ath/ath11k/core.c                       (~2700 lines: device lifecycle, FW load orchestration, per-bus dispatch)
  - drivers/net/wireless/ath/ath11k/pci.c                        (~1320 lines: PCIe transport for discrete cards QCA6390/WCN6855)
  - drivers/net/wireless/ath/ath11k/ahb.c                        (~1500 lines: AHB transport for SoC-integrated IPQ8074/IPQ6018)
  - drivers/net/wireless/ath/ath11k/mhi.c                        (~700 lines: MHI bus for discrete WCN cards on snapdragon platforms)
  - drivers/net/wireless/ath/ath11k/hal.c                        (~3000 lines: HAL — Hardware Abstraction Layer, per-silicon ring/descriptor format)
  - drivers/net/wireless/ath/ath11k/htc.c                        (~600 lines: HTC — Host-Target Communication transport)
  - drivers/net/wireless/ath/ath11k/wmi.c                        (~12000 lines: WMI — Wireless Management Interface, FW cmd protocol)
  - drivers/net/wireless/ath/ath11k/dp.c, dp_rx.c, dp_tx.c       (~10000 lines: DP — Data Path, RX + TX descriptor processing)
  - drivers/net/wireless/ath/ath11k/ce.c                         (~1000 lines: CE — Copy Engine, lightweight cross-CPU msg)
  - drivers/net/wireless/ath/ath11k/dbring.c                     (Debug ring for FW debug data)
  - drivers/net/wireless/ath/ath11k/mac.c                        (~10500 lines: mac80211 op integration)
  - drivers/net/wireless/ath/ath11k/reg.c                        (regulatory)
  - drivers/net/wireless/ath/ath11k/peer.c                       (peer/STA mgmt)
  - drivers/net/wireless/ath/ath11k/spectral.c                   (spectral scan — SS scan + CFR)
  - drivers/net/wireless/ath/ath11k/cfr.c                        (Channel Frequency Response — fine-grained ToF / FTM input)
  - drivers/net/wireless/ath/ath11k/thermal.c                    (thermal mgmt)
  - drivers/net/wireless/ath/ath11k/debugfs.c                    (~1800 lines: debugfs)
  - drivers/net/wireless/ath/ath11k/debugfs_htt_stats.c, debugfs_sta.c
  - drivers/net/wireless/ath/ath11k/coredump.c                   (FW coredump capture)
  - drivers/net/wireless/ath/ath11k/fw.c                         (FW image)
  - drivers/net/wireless/ath/ath11k/qmi.c                        (QMI bus — Qualcomm MSM Interface for FW handshake)
-->

## Summary

ath11k is the Qualcomm Atheros WiFi 6/6E driver, the modern successor to ath10k. Covers Qualcomm silicon: **IPQ8074/IPQ6018** (SoC-integrated AP-class — every modern TP-Link/Linksys/Synology/Asus/Netgear AP router built on Qualcomm), **QCA6390/QCA6391** (discrete cards in laptops + Pixel 5+ phones), **WCN6750/WCN6855** (Snapdragon-X SoC + Pixel 6+ phones + Chromebook), **QCN9074** (extension card for high-end APs). Where iwlwifi is on every Intel laptop, ath11k is on every Qualcomm-based AP router + Snapdragon laptop + Pixel phone. Multi-billion deployed units across phones + AP routers + IoT.

ath11k introduced the "single driver across multiple transports" design — same FW + same code path runs over PCIe (discrete cards) + AHB (SoC-integrated) + MHI (Qualcomm-MSM-PCIe-over-MHI bus on Snapdragon platforms). Transport abstraction in `pci.c`/`ahb.c`/`mhi.c`; core in `core.c`/`hal.c`/`wmi.c`/`dp.c`.

Source: ~61500 lines / 33 .c files. Plus the sibling `ath12k` driver for WiFi 7 (separate Tier-3).

This Tier-3 covers ath11k.

## Rust translation posture

ath11k has a clean multi-transport architecture that translates well:

- **`ath11k-core` crate** — device lifecycle, FW load, mac80211 integration.
- **`ath11k-transport` trait** — implemented by `ath11k-pcie`, `ath11k-ahb`, `ath11k-mhi` sub-crates. Per-transport: probe, MMIO/IRQ setup, RX/TX ring ownership.
- **`ath11k-wmi` crate** — WMI cmd/event protocol. Hundreds of WMI cmds/events; typed builders for each.
- **`ath11k-dp` crate** — Data Path: per-ring descriptor processing. Per-silicon DP version (DP-1/DP-2/DP-3 — IPQ8074 vs QCN9074 vs WCN6855).
- **`ath11k-hal` crate** — per-silicon ring/descriptor format. HAL version distinguishes silicon families.
- **`Ath11kDev<Phase>` typed lifecycle.**

WMI cmd protocol (12K lines in upstream `wmi.c`) is the highest-CVE-density area. Typed cmd/event distinguishes valid combinations + closes parser-bug classes.

Grsec is mandatory. ath11k has had CVE history (CVE-2024-26878-class WMI event handling, CVE-2023-1855-class HAL descriptor parsing).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ath11k_base` | top-level per-device state | `Ath11kDev<Phase>` |
| `struct ath11k` (per-radio) | per-radio state (multi-radio silicon) | `Radio` |
| `struct ath11k_hal` | HAL ring/descriptor format | `Hal<HalVer>` |
| `struct ath11k_wmi_pdev` | WMI per-pdev cmd ctx | `WmiPdev` |
| `struct ath11k_htc_ep` | HTC endpoint | `HtcEp<EpId>` |
| `struct ath11k_dp` | DP ring state | `Dp<DpVer>` |
| `ath11k_pcie_probe(pdev, ent)` / `_remove(pdev)` | PCIe probe | `Pcie::probe` |
| `ath11k_ahb_probe(pdev)` / `_remove(pdev)` | AHB probe (SoC) | `Ahb::probe` |
| `ath11k_mhi_*` | MHI bus integration | `Mhi::*` |
| `ath11k_core_pre_init(...)` / `_init(...)` / `_deinit(...)` / `_qmi_firmware_ready(...)` | core init dispatch | `Core::init` |
| `ath11k_qmi_*` | QMI bus FW handshake | `Qmi::*` |
| `ath11k_fw_init(ab)` / `_fw_destroy(...)` | FW image mgmt | `Fw::*` |
| `ath11k_hal_srng_*` (alloc/setup/access) | HAL SRNG (Service Ring) primitives | `Srng::*` |
| `ath11k_wmi_attach(ab)` / `_detach(...)` | WMI attach | `Wmi::attach` |
| `ath11k_wmi_cmd_send(...)` | WMI cmd send | `Wmi::send_cmd` |
| `ath11k_wmi_*_event(...)` | per-event handlers | `Wmi::on_*_event` |
| `ath11k_wmi_pdev_*` (many — start, stop, get_temperature, etc.) | per-pdev WMI cmds | `WmiPdev::*` |
| `ath11k_wmi_send_init_*` / `_send_setup_complete_*` | init sequence | `Wmi::init_setup` |
| `ath11k_htc_init(ab)` / `_setup_target(...)` / `_send(htc, packet)` | HTC | `Htc::*` |
| `ath11k_ce_setup_*` / `_send_buf(...)` / `_recv_buf(...)` | CE setup + msg | `Ce::*` |
| `ath11k_dp_alloc(ab)` / `_free(ab)` / `_rx_process(...)` / `_tx_send(...)` | DP lifecycle + RX/TX | `Dp::*` |
| `ath11k_dp_htt_*` | DP HTT (HW Translation Table) cmds | `Dp::htt_*` |
| `ath11k_dp_peer_setup(...)` / `_peer_cleanup(...)` | peer setup on DP | `Dp::peer_setup` |
| `ath11k_peer_create(ar, vif, peer_addr)` / `_peer_delete(...)` / `_peer_find(...)` | STA peer mgmt | `Peer::*` |
| `ath11k_mac_op_*` (per-op many) | mac80211 ops | `MacOps::*` |
| `ath11k_mac_op_tx(...)` / `_mac_op_set_key(...)` / `_mac_op_scan_request(...)` | mac80211 main ops | `MacOps::tx` / `_set_key` / `_scan` |
| `ath11k_reg_*` | regulatory | `Reg::*` |
| `ath11k_spectral_*` / `_cfr_*` | spectral / CFR | `Spectral::*` / `Cfr::*` |
| `ath11k_thermal_register(...)` | thermal | `Thermal::register` |
| `ath11k_coredump_*` | coredump capture | `Coredump::*` |
| `ath11k_debugfs_*` | debugfs | `Debugfs::*` |

## Compatibility contract

REQ-1: PCI ID + SoC compatible-string table — IPQ8074, IPQ6018, QCA6390, QCA6391, WCN6750, WCN6855, QCN9074. Frozen.

REQ-2: Multi-transport ABI — PCIe + AHB + MHI all funneled through a common transport trait.

REQ-3: FW image protocol — Qualcomm-signed FW images; loaded via QMI handshake (for MSM platforms) or directly (for IPQ APs).

REQ-4: WMI cmd/event protocol — frozen against `wmi.h` (~5K lines of struct definitions for hundreds of WMI cmds + events).

REQ-5: HTC transport — Host-Target Communication. Multiple endpoints (ctrl, data, etc.).

REQ-6: CE — Copy Engine: lightweight DMA-based msg primitive used by HTC.

REQ-7: DP — Data Path: separate ring system from WMI/HTC; carries RX/TX MPDU data. Per-silicon DP version (different descriptor formats).

REQ-8: HAL SRNG — Service Ring abstraction; per-silicon ring format.

REQ-9: mac80211 integration — full ieee80211_ops registered.

REQ-10: WiFi 6/6E features — OFDMA UL+DL, MU-MIMO UL+DL, TWT, BSS coloring, 6 GHz.

REQ-11: Spectral scan — FFT-based RF spectrum analysis; userspace consumer via debugfs.

REQ-12: CFR — Channel Frequency Response: per-packet I/Q complex sample dump for fine-grained ToF / FTM (802.11mc).

REQ-13: Regulatory — multi-source (NVM, ACPI, user-hint).

REQ-14: Power save — PSM, U-APSD, TWT.

REQ-15: FW coredump — on FW crash; binary blob via devcoredump.

REQ-16: AP-mode — IPQ8074 silicon supports AP mode (BSS-master); used in TP-Link/Linksys APs.

REQ-17: Multi-radio — IPQ8074 silicon has 2 radios (2.4G + 5G) per chip; per-radio state separately.

## Acceptance Criteria

- [ ] AC-1: QCA6390 in laptop probes; WMI ready; mac80211 device registered; iwlist scan works.
- [ ] AC-2: WCN6855 in Pixel/Chromebook — MHI bus + QMI handshake completes; FW ready.
- [ ] AC-3: IPQ8074 in AP router — AHB probe + AP mode; broadcast SSID; STAs associate.
- [ ] AC-4: WPA3-SAE on QCA6390.
- [ ] AC-5: 6 GHz scan + association.
- [ ] AC-6: OFDMA DL on a WiFi 6 AP; per-RU stats observable.
- [ ] AC-7: TWT power save.
- [ ] AC-8: Spectral scan via debugfs; FFT data delivered to userspace.
- [ ] AC-9: CFR for FTM (802.11mc); ToF measurements within ±N meters.
- [ ] AC-10: FW crash recovery: trigger WMI BAD cmd; coredump captured; FW reloaded.
- [ ] AC-11: AP-mode + multi-radio: IPQ8074 with both 2.4G + 5G radios serving STAs simultaneously.

## Architecture

**Multi-transport core.** Bus-agnostic core in `core.c` + `wmi.c` + `dp.c` + `hal.c`. Per-bus glue in `pci.c` / `ahb.c` / `mhi.c`. Transport responsibilities: probe, MMIO map, IRQ setup, CE ring init. Core responsibilities: WMI cmd/event, DP RX/TX, mac80211 integration.

**Init flow (multi-bus).**
1. Bus-glue probes the device.
2. Bus-glue requests QMI handshake (MSM) or directly loads FW (IPQ AP).
3. QMI/direct path completes FW image load.
4. Core init: HTC setup, WMI attach.
5. WMI sends init cmds: set caps, init pdev, enable scheduler.
6. DP attach: alloc rings.
7. mac80211 register.

**WMI cmd protocol.** Cmd from host → FW carries cmd_id + payload. FW processes + responds via WMI event. Events are async (FW-initiated): BSS state changes, scan complete, STA disconnect, regulatory change. ~hundreds of cmd/event types in `wmi.h`.

**HTC + CE.** HTC = Host-Target Communication: typed message channels (multiple endpoints, each its own ring). CE = Copy Engine: lightweight DMA primitive that HTC uses for transport. Each CE channel has a per-direction ring (host→target + target→host).

**DP (Data Path).** Separate ring system from WMI/HTC. Carries per-MPDU RX + TX. Per-silicon descriptor format (DP-1 IPQ8074, DP-2 QCN9074, DP-3 WCN6855). Multi-level rings: RX reaping ring → RX exception ring → RX desc ring → RX msdu ring.

**HAL SRNG.** Service Ring abstraction. Per-ring: head/tail pointers (host + target), entry size, max entries. Per-silicon HAL adapts.

**mac80211 integration.** Full mac80211 ops registered. STA + AP + Mesh + Monitor + P2P. TX from mac80211 → DP TX path; RX from DP → mac80211. Auth/assoc via WMI commands (FW-mediated 802.11 management frame handling).

**QMI handshake (MSM platforms).** On Snapdragon, the WiFi FW is loaded by the platform's MSM not by the WiFi driver directly. QMI bus is the IPC channel. Driver waits for QMI ready signal before WMI attach.

**Spectral + CFR.** Specialized RF analysis features. Spectral: periodic FFT samples streamed via debugfs/relayfs. CFR: per-packet I/Q sample dump used by FTM (802.11mc fine timing measurement) and indoor positioning.

**FW coredump.** On FW crash, capture FW SRAM + register state + recent WMI/HTC msgs into devcoredump-class blob.

## Hardening

- WMI cmd encoder bounded; cmd size capped.
- WMI event parser bounded; per-event-type size validated.
- HAL SRNG ring head/tail consistency invariant.
- DP descriptor parser bounded.
- HTC endpoint count capped at FW caps.
- FW image signature checked via QMI / direct path.
- Spectral / CFR output rate-limited.
- Multi-radio per-radio refcount.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — debugfs writes (spectral config, CFR enable), sysfs writes, regulatory hints bounded.
- **PAX_KERNEXEC** — core/per-bus op tables, HAL ops, WMI cmd dispatch, WMI event dispatch (per-event-id), DP RX/TX dispatch, HTC ops, CE ops, mac80211 ops, AER handlers all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — nl80211 / wireless-ext / debugfs entry paths per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `ath11k_base` refcount, per-radio refcount, per-peer refcount, per-key refcount, per-vif refcount, per-WMI-in-flight refcount saturating.
- **PAX_MEMORY_SANITIZE** — Crypto key material `kfree_sensitive`. FW image load buffer cleared. WMI cmd/response DMA buffers cleared.
- **PAX_UDEREF** — SMAP/SMEP on every user-facing entry.
- **PAX_RAP / kCFI** — core/per-bus/HAL/WMI/HTC/DP/mac80211 op tables all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/ath11k/<bdf>/*`) reveals FW addresses, ring pointers; scrubbed for non-root.
- **GRKERNSEC_DMESG** — verbose ath11k logs (per-WMI-cmd trace, scan results) restricted from non-root.
- **CAP_NET_ADMIN required** — nl80211 / wireless-ext state-changing ops.
- **RF-layer trust boundary explicit** — same as iwlwifi: anything in RF range is adversarial.
- **Per-frame-type RX parser strict** — beacon/probe-response/auth/action frames typed-parsed with bounded length validation. Closes mac80211 BSS-list class via ath11k path.
- **FW image signature mandatory** — Qualcomm-signed via QMI / direct-load path; unsigned refused; LOCKDOWN_INTEGRITY_MAX disables.
- **WMI event opcode allowlist per FW version** — closes "compromised FW sends unknown event triggering parser bug" class. CVE-2024-26878-class defense.
- **WMI event payload length validated per-opcode** — closes parser-OOB class.
- **HAL SRNG head/tail validation in hot path** — descriptor counts derived from SRNG head/tail; validated against ring size before iteration. Closes "compromised FW writes huge head ptr → driver OOB read" class.
- **DP descriptor parser strict** — RX/TX descriptors typed-parsed with field bounds. Closes CVE-2023-1855-class HAL descriptor parsing.
- **QMI msg validation** — QMI messages from platform's MSM validated against expected schemas before consumption. Closes "compromised modem-FW pushes garbage QMI to WiFi driver" cross-domain attack.
- **CFR / Spectral access CAP_SYS_ADMIN** — these features expose fine-grained RF physical-layer data that can be used for side-channels (gait detection, presence sensing); access gated.
- **Multi-radio per-radio isolation** — radios share `ath11k_base` but per-radio state isolated; closes cross-radio state contamination.
- **AP-mode beacon template signature** — when running in AP mode, beacon template installed via WMI cannot include vendor-extension fields that could leak host-side data.
- **TKIP/WEP refused in WPA3-only** — same as iwlwifi.
- **Channel switch + CSA validation** — same as iwlwifi.
- **Regulatory ACPI validation** — same as iwlwifi.
- **Multi-bus transport boundary** — PCIe + AHB + MHI each have separate IRQ + DMA contexts; cross-transport state access refused.
- **FW coredump access CAP_SYS_ADMIN** — coredump may contain in-flight frame data.
- **Spectral relay rate-limit** — Spectral data is high-rate; relay rate-limited per-process to prevent OOM.

Per-doc rationale: ath11k completes the WiFi coverage with the second major WiFi vendor (iwlwifi covers Intel; ath11k covers Qualcomm). The trust posture is identical (RF-layer adversarial), defenses paralleled. Unique to ath11k: multi-bus design (PCIe + AHB + MHI), QMI handshake for Snapdragon, AP-mode + multi-radio for IPQ silicon, Spectral / CFR side-channel concerns (these features can be used for human presence detection / gait analysis — access gated to CAP_SYS_ADMIN). WiFi 6/6E full feature set covered; WiFi 7 in sibling `ath12k` driver (separate Tier-3).

## Open Questions

- [ ] Q1: ath11k vs ath12k convergence — ath12k is the WiFi 7 sibling. Keep separate or unify? Recommendation: separate Tier-3 each (different FW + different silicon).
- [ ] Q2: QMI handshake — depends on platform's MSM availability. On non-Snapdragon (IPQ AP), direct FW load. Document the split path.
- [ ] Q3: AP-mode security — IPQ8074 widely deployed as APs; AP-mode security is critical (beacon-template safety, EAPOL handshake correctness).
- [ ] Q4: Spectral / CFR as side-channels — fine-grained RF data is a recognized privacy/security concern. CAP_SYS_ADMIN may be insufficient — consider an explicit feature gate.
- [ ] Q5: Multi-radio scaling — IPQ8074 has 2 radios; QCN9074 extends to 4 radios. State machine for radio bring-up + tear-down.

## Verification

- **Kani SAFETY**: prove WMI event parser bounds-checked. Prove HAL SRNG head/tail validation.
- **TLA+**: model multi-bus probe + concurrent transport bring-up; check no cross-transport state contamination.
- **Verus**: functional spec of WMI cmd encoder + event parser — encode/decode roundtrip preserves all fields.
- **Kani+Verus**: invariant that every peer has tracked keys; every BA has tracked TID; cleanup at unbind removes all.
- **Integration**: full ath11k matrix — QCA6390 laptop + WCN6855 Chromebook + IPQ8074 AP. WPA3/WPA2/Open. 6 GHz. AP-mode with 100 concurrent STAs. FTM. Spectral scan. FW crash recovery.
- **Fuzz**: beacon/probe-response/auth/action frame fuzzer; WMI event payload fuzzer; DP descriptor fuzzer; QMI msg fuzzer.
- **Penetration**: rogue AP malformed-beacon attack — parser must refuse without crash. Compromised modem-FW QMI msg — driver must refuse.

## Out of Scope

- ath12k (WiFi 7 sibling) — separate Tier-3 (future)
- ath10k (predecessor, WiFi 5) — separate Tier-3 (mostly retired)
- ath9k / ath5k (legacy WiFi-G/N) — separate Tier-3
- QMI bus core — `drivers/soc/qcom/qmi*` separate Tier-3
- MHI bus core — `drivers/bus/mhi/` separate Tier-3
- WPA-Supplicant / hostapd userspace — out of kernel scope
- mac80211 / cfg80211 cores — net/mac80211/ + net/wireless/ separate Tier-3
- Bluetooth coexistence (BTcoex) — covered briefly here, full in btcoex stack
