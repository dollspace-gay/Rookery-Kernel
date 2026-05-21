# Tier-3: drivers/net/wireless/intel/iwlwifi/{iwl-drv,iwl-trans,iwl-nvm-parse,iwl-nvm-utils,iwl-io,iwl-phy-db,iwl-dbg-tlv,iwl-utils}.c + pcie/ + fw/ + cfg/ + mvm/ + mld/ + dvm/ + mei/ — Intel WiFi 6/6E/7 (every Intel-laptop WiFi from AC 7260 through BE200 + per-mode driver: dvm legacy, mvm modern, mld WiFi 7)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/wireless/00-overview.md
upstream-paths:
  - drivers/net/wireless/intel/iwlwifi/iwl-drv.c                  (~2200 lines: module init, fw load, op-mode registration, device probe dispatch)
  - drivers/net/wireless/intel/iwlwifi/iwl-trans.c                (transport layer common — PCIe/MEI/test bus abstraction)
  - drivers/net/wireless/intel/iwlwifi/iwl-nvm-parse.c            (NVM parser — country / channel / capabilities)
  - drivers/net/wireless/intel/iwlwifi/iwl-nvm-utils.c
  - drivers/net/wireless/intel/iwlwifi/iwl-io.c                   (MMIO helpers + force-wake)
  - drivers/net/wireless/intel/iwlwifi/iwl-phy-db.c               (PHY calibration DB)
  - drivers/net/wireless/intel/iwlwifi/iwl-dbg-tlv.c              (debug TLV parser)
  - drivers/net/wireless/intel/iwlwifi/iwl-utils.c
  - drivers/net/wireless/intel/iwlwifi/cfg/                        (per-silicon configs — bz, sc, dr, br, 22000, 9000, 7000, ...)
  - drivers/net/wireless/intel/iwlwifi/fw/                         (~10K+ lines: FW interface — host-cmd, IMG load, dbg, paging, regulatory)
  - drivers/net/wireless/intel/iwlwifi/pcie/                       (PCIe transport — actual MMIO + IRQ + RX/TX rings)
  - drivers/net/wireless/intel/iwlwifi/mvm/                        (~80K lines: mvm op-mode — modern firmware, WiFi 5/6/6E)
  - drivers/net/wireless/intel/iwlwifi/mld/                        (~40K lines: mld op-mode — newest firmware, WiFi 7 + MLO)
  - drivers/net/wireless/intel/iwlwifi/dvm/                        (legacy dvm op-mode — pre-2014 silicon: 4965, 5xx, 6xx, 1000)
  - drivers/net/wireless/intel/iwlwifi/mei/                        (MEI bus integration — Intel Management Engine WiFi sharing for AMT)
  - include/net/mac80211.h, cfg80211.h                              (consumer interfaces — separate Tier-3)
-->

## Summary

iwlwifi is the Intel WiFi driver — covers every Intel WiFi silicon since 2009: the 1000/5000/6000-series 802.11n cards (dvm op-mode legacy), the 7000/8000/9000/22000-series 802.11ac+ax cards (mvm op-mode), and the BE200/BE201 WiFi 7 silicon (mld op-mode). Deployed on tens to hundreds of millions of laptops worldwide. Every Intel-CPU laptop made since 2011 has an Intel WiFi card; Linux laptops + dual-boot setups all run iwlwifi. Modern silicon (AX210/AX211/BE200) supports WiFi 6/6E/7 with full 6 GHz + 320 MHz + Multi-Link Operation.

iwlwifi architecturally has three layers + multiple op-modes:

- **`iwl-drv` core** — module init, FW image parsing, op-mode registration, device probe dispatch by silicon family.
- **`iwl-trans` transport** — bus-agnostic transport layer. PCIe is the primary transport (`pcie/` subdir); MEI is for ME-bridged sharing (`mei/` subdir for vPro/AMT WiFi sharing).
- **`fw/` firmware interface** — FW image load, host-cmd dispatch, IMG/MEM management, regulatory database, debug TLV.
- **Op-mode**: silicon-version-determined; one of:
  - `dvm/` (legacy, 4965/5xx/6xx/1000 silicon — 802.11abgn).
  - `mvm/` (~80K lines, modern, 7000-22000 silicon — 802.11a/b/g/n/ac/ax).
  - `mld/` (~40K lines, newest, BE200+ silicon — 802.11be / WiFi 7 / Multi-Link Operation).

Op-mode is the "personality" of the driver — given the FW image's interface version, the matching op-mode is bound. The op-mode owns mac80211 integration and the per-feature command flow (scan, association, key install, BSS management, A-MPDU, A-MSDU, BA agreements, TID-mapping, EHT/MLO).

This Tier-3 covers ~137000 lines / hundreds of files. Among the largest single-driver trees in the kernel.

## Rust translation posture

iwlwifi is the most architecturally-layered driver Rookery documents. The translation must respect the layering:

- **`iwlwifi-trans-pcie` crate** — PCIe MMIO + IRQ + RX/TX rings + force-wake domains.
- **`iwlwifi-fw` crate** — FW image parse, host-cmd dispatch, IMG/MEM mgmt, regulatory DB, dbg TLV. Used by all op-modes.
- **`iwlwifi-mvm` crate** — modern op-mode for WiFi 5/6/6E (7000-22000 silicon). Largest single op-mode crate.
- **`iwlwifi-mld` crate** — newest op-mode for WiFi 7 + MLO (BE200+).
- **`iwlwifi-dvm` crate** — legacy op-mode for 4965-6xxx.
- **`iwlwifi-mei` crate** — MEI bus shared-access for vPro/AMT WiFi sharing.
- **`iwlwifi-cfg` crate** — per-silicon-family config tables.

Phase typing: `Iwl<Probed → FwLoaded → OpModeBound → Online>`. Op-mode bound only when FW is loaded + silicon-family matches.

Host commands typed: each cmd_id a method on `Hcmd`. FW image structure typed (parsed at load).

The grsec/PaX section is mandatory. iwlwifi has had many CVEs (CVE-2022-41674 mac80211 BSS frame OOB read via iwlwifi RX, CVE-2022-42721 BSS list corruption, CVE-2024-26845 WPA-handshake parsing). WiFi drivers are particularly bug-prone because they parse attacker-controlled radio frames; the trust boundary is "anything within RF range can send malformed beacons/probe-responses/auth frames".

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iwl_drv` | top-level driver instance | `IwlDrv<Phase>` |
| `struct iwl_trans` | transport abstraction | `Trans<BusKind>` |
| `struct iwl_op_mode` | op-mode binding | `OpMode<Mode>` (dvm/mvm/mld) |
| `struct iwl_fw` | parsed FW image | `Fw` |
| `struct iwl_cfg` | per-silicon config | `CfgInfo` |
| `struct iwl_mvm` (or _mld, _dvm) | per-op-mode state | `Mvm` / `Mld` / `Dvm` |
| `iwl_drv_start(...)` / `_stop(...)` | drv lifecycle | `IwlDrv::start` / `stop` |
| `iwl_request_firmware(drv, first)` / `_req_fw_callback(...)` | FW load | `Fw::load` |
| `iwl_drv_match_op_mode(...)` | op-mode selection by FW interface ver | `OpMode::match` |
| `iwl_op_mode_*_ops` (per-mode) | op-mode vtable | `OpModeOps` trait |
| `iwl_trans_pcie_alloc(...)` / `_free(...)` | PCIe transport alloc | `TransPcie::alloc` / `Drop` |
| `iwl_trans_pcie_send_hcmd(...)` | host-cmd send via PCIe | `TransPcie::send_hcmd` |
| `iwl_trans_start_hw(...)` / `_op_mode_leave(...)` | start hw + op-mode leave | `Trans::start_hw` |
| `iwl_trans_pcie_rx(*)` | RX path | `TransPcie::rx` |
| `iwl_trans_pcie_tx(*)` | TX path | `TransPcie::tx` |
| `iwl_pcie_apm_init(trans)` / `_apm_stop(...)` | APM (Power Management) bring-up | `Apm::init` / `stop` |
| `iwl_pcie_irq_handler(...)` | top-level IRQ | `Irq::handler` |
| `iwl_pcie_rx_handle(...)` / `_irq_msix_handler(...)` | RX dispatcher | `Rx::handle` |
| `iwl_mvm_rx_rx_mpdu(...)` / `_rx_phy_cmd(...)` / `_rx_beacon_notif(...)` (many) | mvm RX path per-packet-type | `MvmRx::*` |
| `iwl_mvm_send_cmd(...)` / `_send_cmd_pdu(...)` | mvm host-cmd send | `MvmHcmd::send` |
| `iwl_mvm_scan_size(...)` / `_scan_setup_scan_req(...)` / `_scan_start(...)` | scan path | `Scan::*` |
| `iwl_mvm_mac_setup_register(mvm)` | mac80211 register | `Mvm::mac80211_register` |
| `iwl_mvm_mac_tx(...)` / `_mac_offchannel_tx(...)` | TX entry from mac80211 | `Mvm::tx` |
| `iwl_mvm_add_key(...)` / `_remove_key(...)` | crypto-key install | `Crypto::add_key` |
| `iwl_mvm_add_sta(...)` / `_rm_sta(...)` | station mgmt | `Sta::add` / `rm` |
| `iwl_mvm_mac_ampdu_action(...)` | A-MPDU control | `Ampdu::action` |
| `iwl_mld_*` | mld op-mode equivalents | `MldOps::*` |
| `iwl_dvm_*` | dvm op-mode equivalents (legacy) | `DvmOps::*` |
| `iwl_fw_dbg_collect_*` | FW debug dump | `FwDbg::*` |
| `iwl_force_nmi(trans)` | force FW NMI for recovery | `Trans::force_nmi` |
| `iwl_pcie_d3_handshake(...)` / `_d3_complete_suspend(...)` | D3 PM handshake | `Pm::*` |
| `iwl_mei_register(...)` / `_unregister(...)` | MEI bus integration | `Mei::*` |
| `iwl_acpi_get_wifi_pkg(...)` | ACPI WiFi data (wRDD/wRDS/etc.) | `Acpi::*` |
| `iwl_drv_eeprom_init_*` | EEPROM/NVM parse | `Nvm::init` |
| `iwl_regulatory_*` | regulatory limits | `Regulatory::*` |

## Compatibility contract

REQ-1: PCI ID table — every Intel WiFi silicon: 1000/5xxx/6xxx (dvm), 7260/3160/7265/8260/8265/9000/9260/AX200/AX201/AX210/AX211/BE200/BE201. Frozen.

REQ-2: FW image format — Intel FW container format with TLV sections (op-mode version, paging tables, debug data, regulatory). Parsed by `iwl-drv`.

REQ-3: Op-mode binding — dvm / mvm / mld selected based on FW interface version. Frozen against known FW versions.

REQ-4: Transport ABI — PCIe transport + MEI transport. PCIe is the production path; MEI is for vPro/AMT shared mode.

REQ-5: mac80211 integration — full mac80211 interface (driver registers `ieee80211_ops`); covers STA + AP + Mesh + IBSS + Monitor + P2P + NAN.

REQ-6: WiFi 6/6E features — 1024-QAM, OFDMA, MU-MIMO (UL+DL), TWT, BSS coloring, 6 GHz channels (UNII-5/6/7/8).

REQ-7: WiFi 7 features (mld) — 4096-QAM, 320 MHz, Multi-Link Operation (MLO — single STA on multiple bands simultaneously), Multi-Link Setup, Multi-AP coordination.

REQ-8: A-MPDU + A-MSDU — aggregation per spec; per-TID BA agreements.

REQ-9: Power-save — PSM, U-APSD, TWT (WiFi 6+), runtime PCIe D3hot/D3cold during idle.

REQ-10: Scan — passive + active + scheduled scan; 2.4G/5G/6G channel mask per regulatory.

REQ-11: Crypto offload — WEP (legacy), TKIP (deprecated), CCMP-128/256, GCMP-128/256. WPA2 + WPA3 + 802.1X coexistence.

REQ-12: Regulatory — multi-source (NVM, ACPI wRDD/wRDS/wTDS, mac80211 user-hint). World regulatory database per FW.

REQ-13: D3 (suspend) wake-on-WLAN — magic packet, GTK rekey, EAPOL, optional NS proxy.

REQ-14: ACPI integration — wRDD (Wireless Regulatory Data), wRDS, wGDS, wPPD for per-system regulatory + power tables.

REQ-15: MEI bus (vPro/AMT) — share WiFi between host OS and ME firmware (out-of-band management).

REQ-16: 11k/v/r — neighbor reports, BSS transition, fast BSS transition.

REQ-17: FW debug — TLV-based debug data collection; coredump on FW exception.

REQ-18: PCIe MSI-X — modern silicon uses MSI-X with per-CPU RX vectors.

## Acceptance Criteria

- [ ] AC-1: AX210 probes; FW loads; mvm op-mode binds; mac80211 device registered; `iwlist scan` works.
- [ ] AC-2: WPA3-SAE connection to a WPA3-Personal AP; encrypted traffic.
- [ ] AC-3: 6 GHz channel scan + association on AX210 in a 6 GHz-permitted region.
- [ ] AC-4: BE200 with mld op-mode; WiFi 7 MLO with a compatible AP; dual-band STA via MLO.
- [ ] AC-5: A-MPDU aggregation working; per-TID BA stats observable.
- [ ] AC-6: D3 suspend + WoWLAN magic-packet wake.
- [ ] AC-7: Scheduled scan in background; battery impact minimal.
- [ ] AC-8: vPro/AMT integration via MEI: host OS + ME share WiFi cleanly.
- [ ] AC-9: FW crash recovery: trigger FW NMI; coredump captured; FW reloaded; reconnect.
- [ ] AC-10: Regulatory ACPI: WiFi card respects per-system ACPI regulatory restrictions.
- [ ] AC-11: TWT: power-save with TWT-aware AP; observable lower idle current.

## Architecture

**Three-layer + op-mode.** `iwl-drv` orchestrates. `iwl-trans-pcie` handles PCIe MMIO + IRQ + RX/TX rings. `fw/` parses FW images + handles host-cmd. Op-mode (`mvm/`, `mld/`, `dvm/`) provides the mac80211-facing logic.

**FW load + op-mode binding.** At probe:
1. iwl-drv determines silicon family from PCI ID; selects FW filename + cfg table.
2. `request_firmware` async.
3. On FW load, parse TLV container, extract op-mode interface version.
4. Bind matching op-mode (`mvm` for 7000-22000, `mld` for BE200+).
5. Op-mode init: register with mac80211; bring up internal state.

**Host commands.** Driver-to-FW communication via host-cmd messages. Each cmd has opcode + payload. Synchronous (poll for response) or async (callback on response). Per-version ABI varies — newer FW versions add/remove cmds.

**RX path (mvm).** Per-MSI-X RX vector → ring poll → per-packet dispatch via `iwl_mvm_rx_rx_mpdu` (data), `_rx_phy_cmd` (PHY metadata), `_rx_beacon_notif` (beacons), `_rx_*_notif` (per-type notifications). MPDU data delivered to mac80211 via `ieee80211_rx_napi`.

**TX path (mvm).** mac80211 calls `iwl_mvm_mac_tx`; driver builds host cmd with frame data; FW transmits; TX completion via async notification.

**Crypto offload.** Per-key install via `iwl_mvm_add_key`. FW does CCMP/GCMP/TKIP/WEP. Driver passes key material once; FW holds. Key material kfree_sensitive after install.

**Scan.** Driver issues SCAN host cmd; FW schedules across channels per regulatory; results posted via SCAN_RESULTS notification. Scheduled scan: similar, with periodic intervals.

**Power save.** PSM (Power Save Mode) — STA periodically wakes for beacons. U-APSD — async power save. TWT (WiFi 6+) — pre-negotiated wake intervals.

**D3 (suspend) wake-on-WLAN.** Driver programs WoWLAN patterns into FW; FW continues to listen during S3. Wake conditions: magic packet, EAPOL, GTK rekey, NS proxy.

**MEI bus integration.** vPro/AMT shares the WiFi between host OS and ME. iwlwifi-mei coordinates ownership transitions via the MEI bus.

**FW crash recovery.** FW exception triggers coredump; driver reloads FW + re-init op-mode. State that depended on FW (associations, BA agreements, keys) is rebuilt.

## Hardening

- FW image TLV parser bounded.
- Host-cmd response timeout enforced.
- Per-RX-packet headers validated before passing to mac80211.
- Crypto key material zeroed after FW install.
- Scan request parameters validated.
- WoWLAN patterns count + size capped.
- ACPI table parsers bounded.
- Op-mode lifetime: must unbind cleanly on FW reload.
- MEI bus ownership transitions atomic.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — debugfs writes (FW load, debug TLV), sysfs writes, regulatory hint writes all bounded.
- **PAX_KERNEXEC** — `iwl_drv` ops, op-mode vtables, transport ops, host-cmd dispatch, RX packet-type dispatch, ACPI parsers, MEI ops, `pci_error_handlers`, mac80211 ops all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — nl80211 / wireless-ext / sysfs entry paths per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `iwl_drv` refcount, per-op-mode refcount, per-STA refcount, per-key refcount, per-BA-agreement refcount saturating.
- **PAX_MEMORY_SANITIZE** — Crypto key material `kfree_sensitive`. WoWLAN pattern buffer cleared after S3 resume. FW image load buffer cleared after FW install. ACPI table copy buffers cleared.
- **PAX_UDEREF** — SMAP/SMEP on every user-facing entry.
- **PAX_RAP / kCFI** — `iwl_drv` ops, op-mode vtables, transport ops, host-cmd dispatch (per-cmd-id), RX packet-type dispatcher (per-pkt-id), ACPI parser callbacks, MEI bus ops all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/iwlwifi/<bdf>/*`) reveals FW image addresses, ring pointers, FW debug regions; scrubbed for non-root.
- **GRKERNSEC_DMESG** — iwlwifi verbose logs (scan results revealing nearby networks, per-packet RX trace) restricted from non-root.
- **CAP_NET_ADMIN required** — nl80211 ops that change config (scan, associate, key install, channel set, regulatory hint).
- **RF-layer trust boundary explicit** — anything within RF range can send malformed beacons / probe responses / auth frames. Driver must defensively parse every byte.
- **RX frame parser strict per-frame-type** — beacon, probe-response, association-response, auth, action frames each have a typed parser with bounded length validation. Closes CVE-2022-41674 / -42721 / -42722 / -42723 / -42724 / -42725 / -42726 class (mac80211 BSS-list corruption from malformed iwlwifi RX).
- **FW image signature mandatory** — Intel-signed FW images; LOCKDOWN_INTEGRITY_MAX disables FW updates.
- **Host-cmd opcode allowlist per FW version** — only opcodes known to the bound op-mode + FW version accepted; closes "compromised FW returns unknown cmd response triggering parser bug" class.
- **Host-cmd response length validated per opcode** — closes parser-OOB class.
- **Regulatory ACPI table validation** — wRDD/wRDS/etc. tables validated against expected layouts before consumption. Closes "ACPI BIOS exposes malformed wRDD → driver UB" class.
- **MEI bus ownership transitions atomic** — only one side (host OS or ME) owns the WiFi at any time; transitions through MEI bus protocol. Closes a path where the ME could pre-empt host OS WiFi mid-flight, causing UAF.
- **Scan rate-limit** — active scans rate-limited per process (closes "battery drain via scan spam"); passive scans rate-limited too.
- **WoWLAN pattern count capped** — per-WiFi-card WoWLAN pattern count limited; closes a path where a malicious pre-suspend process fills the pattern table.
- **Crypto key material kfree_sensitive after FW install** — keys live in FW SRAM; host RAM copy cleared immediately.
- **FW debug-collection CAP_SYS_ADMIN** — debug dumps may contain in-flight frame data; coredump access gated.
- **Regulatory user-hint CAP_NET_ADMIN + LOCKDOWN bypass-disabled when LOCKDOWN_INTEGRITY_MAX** — closes "user-space lies about regulatory domain to bypass Tx-power limits" class.
- **Per-STA TKIP / WEP refused in WPA3-only mode** — when AP advertises WPA3 only, refuse TKIP/WEP keys (defends against downgrade attacks).
- **Multi-Link Setup (WiFi 7) per-link auth state isolation** — each MLO link's auth state separate; cross-link injection refused.
- **TWT negotiation parameter validation** — TWT wake intervals validated against silicon limits; out-of-range refused.
- **MEI WiFi sharing CAP_SYS_ADMIN** — enable/disable MEI WiFi sharing requires CAP_SYS_ADMIN.
- **ACPI WiFi DSM allowlist** — only the known wDSM functions for WiFi accepted; unknown refused.
- **Channel switch announcement (CSA) action frame source-MAC validation** — closes "spoofed CSA from rogue AP forces channel change" class (a known WiFi attack).

Per-doc rationale: WiFi drivers are the highest-risk attack surface in the kernel because the trust boundary is "anything in RF range". Any malformed beacon / probe-response / auth frame transmitted by an adversary's radio gets parsed by mac80211 + iwlwifi. Historical CVE concentration: 2022 saw 6+ CVEs in mac80211 BSS-list parsing triggered by malformed beacons from rogue APs (CVE-2022-41674 + family). Defenses: per-frame-type typed parsers with bounded length validation close the class structurally. FW image signature + LOCKDOWN integration close FW substitution. Crypto key kfree_sensitive closes key-leak. Regulatory ACPI validation closes BIOS-induced UB. MEI bus ownership transitions atomic close the vPro/AMT cross-domain UAF. WiFi 7 MLO per-link auth isolation closes cross-link injection (a new class introduced with WiFi 7).

## Open Questions

- [ ] Q1: dvm legacy op-mode — pre-2014 silicon. Maintain or drop? Recommendation: maintain via `iwlwifi-dvm` crate; silicon is in long-life embedded.
- [ ] Q2: mld vs mvm convergence — mld is newer; will mvm be retired? Recommendation: keep mvm for installed base.
- [ ] Q3: MEI bus integration — niche but production-critical for enterprise vPro deployments. Cover, but isolate behind a feature flag.
- [ ] Q4: WiFi 7 MLO scale — multi-link bring-up + tear-down has many edge cases. Need explicit state-machine modeling.
- [ ] Q5: Regulatory ACPI tables — Intel-defined per-system tables; vendor-specific. Keep an allowlist of known table layouts.
- [ ] Q6: Mesh + IBSS + P2P — less-deployed modes. Cover minimally; defer deep validation.

## Verification

- **Kani SAFETY**: prove FW image TLV parser cannot OOB-read. Prove host-cmd response length validation cannot be bypassed.
- **TLA+**: model MEI bus ownership transitions + concurrent host-OS scan requests; check no UAF during transition.
- **Verus**: functional spec of the per-frame-type RX parser — for valid frame, extracts fields; for malformed (any field oversized, truncated, etc.), returns -EINVAL.
- **Kani+Verus**: invariant that every BA agreement has a tracked TID; every STA has tracked keys; cleanup at unbind removes all.
- **Integration**: AX210 + BE200 connection matrix across WPA2/WPA3/WPA3-Enterprise/Open APs; 6 GHz scan + association; WiFi 7 MLO with compatible AP; D3 + WoWLAN cycle; scheduled scan battery soak. FW crash injection + recovery.
- **Fuzz**: beacon frame fuzzer (mutate every field); probe-response fuzzer; auth frame fuzzer; host-cmd response fuzzer (FW-side mock); ACPI wRDD table fuzzer.
- **Penetration**: from a rogue AP, send malformed CSA action frames — driver must validate source MAC + refuse from non-associated source. Send malformed beacon at boundary lengths — must reject without crash.

## Out of Scope

- mac80211 core (`net/mac80211/`) — separate Tier-3 (pre-existing in net/)
- cfg80211 core (`net/wireless/`) — separate Tier-3 (pre-existing)
- WPA-Supplicant userspace — out of kernel scope
- per-cfg silicon data tables — covered briefly in cfg/ subdir, full coverage per-silicon
- per-silicon erratas — covered in cfg/
- ucode FW source — Intel-proprietary, out of scope
- iwlmei (auxiliary tools) — out of kernel scope
- Older `iwlegacy` driver (4965/3945) — separate Tier-3 if needed (mostly retired)
