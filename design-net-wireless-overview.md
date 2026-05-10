---
title: "Tier-2: net/wireless — Wireless networking (cfg80211 + mac80211 + nl80211 UAPI + regulatory + per-driver iwlwifi/ath/mt76/rtw/brcmfmac/marvell/realtek)"
tags: ["tier-2", "net", "wireless", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the Linux wireless networking stack — every Wi-Fi card, every USB Wi-Fi dongle, every embedded SoC Wi-Fi (smartphone, tablet, laptop), every Wi-Fi 6/6E/7 access-point implementation, every monitor-mode pcap capture, every wpa_supplicant/iwd/hostapd consumer, every IEEE 802.11ax/ay/be deployment in Linux. Three concentric scopes:

- **cfg80211 core** (`net/wireless/`): the userspace ABI gateway via NL80211 netlink + the wireless extension legacy ABI. Owns regulatory database (per-country channel/power tables loaded from CRDA / kernel-builtin), wiphy registration, per-vif management, scan/connect state machine, BSS database, mesh, IBSS, OCB (Outside-Context-of-BSS for V2X), AP/P2P/NaN, PMSR (Peer-to-Peer Measurement — FTM 802.11mc), TDLS (Tunneled Direct Link Setup), 6 GHz band gating, sysfs/debugfs surface, certs (regulatory database signing key)
- **mac80211** (`net/mac80211/`): the Soft-MAC implementation underneath every "soft-MAC" Wi-Fi driver — consumes from cfg80211, implements 802.11 station/AP MLME, link-layer state machines, encryption (WEP/TKIP/CCMP/GCMP/AES-CMAC/AES-GMAC), 802.11n/ac/ax/be aggregation (A-MPDU/A-MSDU), block-ack, retry/rate-control, AP-side TIM/DTIM/PSM, mesh-routing (HWMP), IBSS, OCB, P2P listen + GO, A-TIM/SP, MLO (802.11be Multi-Link Operation), Multi-BSSID, FILS, FT-OTA, OWE, SAE/PMK caching, EAP-Hand-Shake key install, TWT scheduling, airtime fairness, MU-MIMO, OFDMA, BSS color, EHT, 320 MHz (Wi-Fi 7), 6 GHz preamble puncturing
- **per-driver wireless drivers** (`drivers/net/wireless/`): per-vendor LLDs, both **soft-MAC** (consume mac80211 — most modern drivers) and **full-MAC** (implement MLME entirely in firmware — historically Broadcom, some Realtek, some Qualcomm Atheros)

Heavily cross-referenced from `net/00-overview.md` (every wireless netdev consumer there), `drivers/net/ethernet/00-overview.md` (page-pool + napi shared), `drivers/pci/00-overview.md` (most internal Wi-Fi cards are PCIe-MiniPCIe-M.2), `drivers/usb/00-overview.md` (USB Wi-Fi dongles), `drivers/sdio/` (SDIO Wi-Fi for embedded), `crypto/00-overview.md` (CCMP/GCMP/CMAC/GMAC AEAD), `kernel/cgroup/blkio.md` (no — wireless lives under net not blkio), future `net/bluetooth/00-overview.md` (Bluetooth is sibling subsystem; some chips share radios with Wi-Fi), `drivers/iommu/00-overview.md` (PCIe Wi-Fi DMA), `kernel/dma/00-overview.md` (page-pool integration for newer drivers).

### Out of Scope

- Per-Tier-3 (Phase D) — per-driver docs arrive incrementally
- Most legacy wireless drivers (`admtek/`, `atmel/`, `intersil/`, `purelifi/`, `quantenna/`, `ralink/`, `silabs/`, `st/`, `ti/`, `zydas/`, etc.) — keep present compile-gated off
- ARM-only embedded Wi-Fi (most SoC-integrated wcn36xx outside Snapdragon laptops)
- WiMAX (`net/wimax/` — separately removed upstream)
- 32-bit-only paths
- Implementation code

### components

### cfg80211 (`net/wireless/`)

- **core** (`core.c` + `core.h`): wiphy + wdev (wireless-device) lifecycle, netdev notifier glue
- **nl80211** (`nl80211.c` + `nl80211.h`): the modern UAPI — NETLINK_GENERIC family `nl80211` with ~200 commands + ~400 attribute types
- **wext** (`wext-*`): legacy Wireless Extensions ioctl ABI (`SIOCSIWFREQ`, `_SIWMODE`, `_SIWESSID`, etc.); deprecated but kept for legacy iwconfig/iwlist tools
- **scan** (`scan.c`): scan request management + BSS database
- **mlme** (`mlme.c`): cross-driver MLME helpers (auth/assoc/deauth)
- **chan / chan-trace** (`chan.c` + `chan_trace.c`): channel definition + per-channel state
- **reg** (`reg.c`): regulatory database — per-country channel/power tables loaded from kernel-builtin DB + CRDA userspace + per-driver regulatory hints; `cfg80211_get_regdom`
- **regdb-certs** (`certs/`): regulatory database signing keys (CRDA + Sforshee key)
- **debugfs** (`debugfs.c` + `debugfs.h`): `/sys/kernel/debug/ieee80211/`
- **trace** (`trace.h` + `trace.c`): tracepoints `cfg80211:*`
- **ibss** (`ibss.c`): IBSS (ad-hoc) mode
- **mesh** (`mesh.c`): 802.11s mesh mode
- **ocb** (`ocb.c`): OCB (Outside Context of BSS) for V2X
- **ap** (`ap.c`): AP-mode helpers
- **pmsr** (`pmsr.c`): Peer-to-Peer Measurement — FTM 802.11mc (Fine-Timing-Measurement for Wi-Fi positioning)
- **radiotap** (`radiotap.c`): radiotap header generation/parsing for monitor-mode pcap capture
- **ethtool** (`ethtool.c`): ethtool integration for wireless
- **of** (`of.c`): OpenFirmware/DT helpers (mostly ARM)
- **michael-mic** (`michael-mic.c`): WPA1 Michael MIC (legacy TKIP)
- **rdev-ops.h**: per-wiphy rdev_ops dispatch macros
- **util** (`util.c`): IE parsers + cipher-suite helpers
- **wext-{compat,priv,sme,spy}**: WEXT compat + private ioctl + sme + IW spy/STAT helpers
- **lib80211**: shared per-cipher helpers
- **sysfs** (`sysfs.c`): per-wiphy sysfs at `/sys/class/ieee80211/phy<N>/`

### mac80211 (`net/mac80211/`)

- **main** (`main.c` + `iface.c` + `chan.c` + `cfg.c`): wiphy registration with cfg80211; per-vif lifecycle; per-channel context binding
- **mlme** (`mlme.c`): station-mode MLME — auth/assoc/deauth/disassoc/deauth + 4-way handshake support
- **rx / tx** (`rx.c` + `tx.c`): per-frame ingress + egress paths (the hottest paths)
- **sta_info** (`sta_info.c`): per-station state, per-tid state, A-MPDU + A-MSDU + block-ack handles
- **agg-rx / agg-tx** (`agg-rx.c` + `agg-tx.c`): A-MPDU aggregation receive + transmit
- **key** (`key.c`): per-link key install/uninstall + cipher dispatch
- **wpa** (`wpa.c`): WPA/WPA2/WPA3 hooks
- **scan** (`scan.c`): scan execution
- **mesh*** (`mesh*.c`): 802.11s mesh — HWMP routing, mesh peering, mesh path table
- **ibss** (`ibss.c`): IBSS-mode MLME
- **ocb** (`ocb.c`): OCB-mode (V2X)
- **fils_aead** (`fils_aead.c`): FILS (Fast Initial Link Setup) AEAD
- **ht** (`ht.c`): 802.11n HT helpers
- **vht** (`vht.c`): 802.11ac VHT helpers
- **he** (`he.c`): 802.11ax HE (high-efficiency) helpers
- **eht** (`eht.c`): 802.11be EHT (extremely-high-throughput) helpers
- **mlo** (`mlo.c`): 802.11be Multi-Link Operation
- **rate / rc80211_minstrel_ht** (`rate.c` + `rc80211_minstrel_ht.c` + `rc80211_minstrel_ht_debugfs.c`): rate-control framework + minstrel-HT default rate-control algorithm
- **airtime / airtime_policy** (`airtime.c`): airtime-fairness + airtime queue limit
- **status** (`status.c`): per-frame Tx status reporting + retry tracking
- **led** (`led.c`): per-radio LED triggers
- **debugfs / debugfs_key / debugfs_netdev / debugfs_sta** (`debugfs*.c`): `/sys/kernel/debug/ieee80211/phy<N>/`
- **driver-ops** (`driver-ops.h`): per-driver `ieee80211_ops` dispatch macros
- **aead_api / aes_*** (`aead_api.c` + `aes_ccm.h` + `aes_cmac.c` + `aes_gcm.h` + `aes_gmac.c`): per-cipher AEAD glue
- **michael_mic** (`michael-mic.c`): WPA1 Michael MIC
- **ethtool** (`ethtool.c`): ethtool integration
- **trace / trace_msg** (`trace.h` + `trace.c` + `trace_msg.h`): tracepoints `mac80211:*`
- **vendor** (`vendor.c`): per-vendor private nl80211 commands

### Per-driver wireless drivers (`drivers/net/wireless/`)

#### Active maintenance set in v0

| Driver | Path | Coverage |
|---|---|---|
| **iwlwifi** | `intel/iwlwifi/` | Intel Wi-Fi 6/6E/7 (AX200/AX201/AX210/AX211/BE200/BE201) — soft-MAC; the dominant Intel laptop Wi-Fi |
| **iwlegacy** | `intel/iwlegacy/` | Intel 3945/4965/5000-series (legacy; keep) |
| **ipw2100/2200** | `intel/ipw2x00/` | Intel pre-Centrino (legacy; compile-gated off candidate) |
| **ath9k** | `ath/ath9k/` | Atheros AR9000-series (802.11n; soft-MAC) |
| **ath10k** | `ath/ath10k/` | Qualcomm Atheros 802.11ac (QCA988X/QCA9377/QCA6174 — soft-MAC + offloaded MLME) |
| **ath11k** | `ath/ath11k/` | Qualcomm 802.11ax (QCA6390/WCN6855/QCN9074 — Wi-Fi 6/6E) |
| **ath12k** | `ath/ath12k/` | Qualcomm 802.11be (QCN9274 — Wi-Fi 7) |
| **ath6kl** | `ath/ath6kl/` | Atheros AR6003 SDIO (legacy embedded) |
| **carl9170** | `ath/carl9170/` | Atheros AR9170 USB (legacy) |
| **wcn36xx** | `ath/wcn36xx/` | Qualcomm WCN36xx (Snapdragon SoC integrated) |
| **mt76** | `mediatek/mt76/` | MediaTek Wi-Fi 5/6/6E/7 (MT76x0/x2/x8/x9, MT7615, MT7663, MT7915, MT7921, MT7925, MT7990 — Wi-Fi 7) |
| **mwifiex** | `marvell/mwifiex/` | Marvell 8786/8787/8797/8897/8997 (consumer/embedded; soft-MAC + full-MAC hybrid) |
| **mwl8k** | `marvell/mwl8k/` | Marvell 88w8xxx (legacy AP) |
| **libertas** + **libertas_tf** | `marvell/libertas*/` | Marvell legacy |
| **brcmfmac** | `broadcom/brcm80211/brcmfmac/` | Broadcom FullMAC — the dominant non-Intel laptop Wi-Fi (43xx + 43xxx + 43xxxx + Cypress CYW43xxx) |
| **brcmsmac** | `broadcom/brcm80211/brcmsmac/` | Broadcom soft-MAC (legacy 43224 / 43225) |
| **rtw88** | `realtek/rtw88/` | Realtek RTL8822BE/RTL8822CE/RTL8723DE/RTL8723DU (consumer Wi-Fi 5) |
| **rtw89** | `realtek/rtw89/` | Realtek RTL8852AE/RTL8852BE/RTL8852CE/RTL8851BE (Wi-Fi 6/6E/7) |
| **rtl8xxxu** | `realtek/rtl8xxxu/` | Realtek USB (RTL8188/RTL8192/RTL8723) |
| **rtl818x** | `realtek/rtl818x/` | Realtek 8180/8185/8187 (legacy) |
| **rtlwifi** | `realtek/rtlwifi/` | Realtek RTL8188/8192/8723 PCIe (legacy) |
| **rsi** | `rsi/` | Redpine Signals RS9113/RS9116 (industrial; SDIO/USB/SPI) |
| **mwifiex** + **libertas** | `marvell/` | Marvell consumer (above) |
| **virtual mac80211** + **mac80211_hwsim** | `virtual/mac80211_hwsim.c` | mac80211 software-simulator (used heavily for testing + CI) |

#### Compile-gated-off / out of v0

Legacy + niche: `admtek/`, `atmel/`, `intersil/{p54, prism54, hostap, orinoco}`, `microchip/wilc1000`, `purelifi/`, `quantenna/`, `ralink/{rt2x00}`, `silabs/{wfx}`, `st/{cw1200}`, `ti/{wlcore, wl1251, wl12xx, wl18xx}`, `zydas/{zd1201, zd1211rw}`, `prism54/`, `airo*`, `arlan-*`, `b43/`, `b43legacy/`, `cisco-airo`, `hermes/`, `ipw2100/`, `ipw2200/`, `iwl3945/`, `iwl4965/`, `lib80211_crypt_*` (legacy WEP/TKIP-only).

### scope

This Tier-2 governs:
- `/home/doll/linux-src/net/wireless/` (~30 files)
- `/home/doll/linux-src/net/mac80211/` (~80 files)
- `/home/doll/linux-src/drivers/net/wireless/` (~20 vendor subdirs covering thousands of files)
- Public headers `include/net/{cfg80211, mac80211, regulatory, ieee80211_radiotap}.h`
- UAPI `include/uapi/linux/nl80211.h` + `include/uapi/linux/wireless.h` + `include/uapi/linux/wireless_radiotap.h`

### compatibility contract — outline

### NETLINK_GENERIC `nl80211` family

UAPI byte-identical so wpa_supplicant + iwd + hostapd + iw + iwctl + ModemManager + NetworkManager + ConnMan consume unchanged. ~200 NL80211_CMD_* commands + ~400 NL80211_ATTR_* attributes covering wiphy mgmt, scan/connect, AP setup, mesh setup, monitor mode, regulatory hints, P2P/NaN, PMSR/FTM, TDLS, mlo, key install, statistics, vendor cmds.

### Wireless Extensions (legacy ABI)

`SIOCSIWFREQ`, `_GIWFREQ`, `_SIWMODE`, `_GIWMODE`, `_SIWESSID`, `_GIWESSID`, `_SIWENCODE`, `_GIWENCODE`, `_SIWAP`, `_GIWAP`, etc. byte-identical so legacy `iwconfig` + `iwlist` + `iwpriv` consume unchanged (deprecated but kept).

### sysfs surface

Per-wiphy `/sys/class/ieee80211/phy<N>/`:
- `address`, `addresses`, `name`, `index`, `device/`, `macaddress`, `country`, `register_only`

Per-vif netdev `/sys/class/net/<wlan>/`:
- standard netdev attrs + `wireless/` subdir with link-layer stats

Per-station `/sys/kernel/debug/ieee80211/phy<N>/netdev:<wlan>/stations/<MAC>/{ht_capa, vht_capa, he_capa, eht_capa, signal, signal_avg, tx_*, rx_*, ...}`

Layout + content byte-identical so `iw dev <wlan> station dump` + `iw phy phy<N> info` consume unchanged.

### Regulatory

Kernel-builtin regulatory DB + CRDA userspace fallback. Per-country channel + tx-power table loaded from `/lib/firmware/regulatory.db` (signed by Sforshee key). Behavior identical so per-country channel restrictions + 6 GHz LPI/SP/VLP gating apply identically.

### `/sys/kernel/debug/ieee80211/`

Per-wiphy debugfs:
- `aqm` (Airtime Queue Mgmt parameters), `frame_drop_count`, `force_tx_status`, `aql_*`, `airtime_flags`, `aql_pending`, `power`, `total_ps_buffered`, `wake_tx_queue`, etc.
- Per-station + per-key + per-link debugfs

Format byte-identical for in-tree consumers + iw debug commands.

### Tracepoints

`events/{cfg80211, mac80211, iwlwifi, ath, mt76, brcmfmac, rtw88, rtw89}/` byte-identical names + format.

### Per-driver firmware

Most Wi-Fi drivers load firmware from `/lib/firmware/{iwlwifi-*, ath10k/*, ath11k/*, ath12k/*, mt7921/*, mt7925/*, brcm/*, rtw88/*, rtw89/*}`. Loading via `request_firmware` (cross-ref `drivers/base/firmware-loader.md`).

### Module params per-driver

- `iwlwifi.power_save`, `iwlwifi.disable_11ax`, `iwlwifi.disable_11be`, `iwlwifi.uapsd_disable`, `iwlwifi.led_mode`, `iwlwifi.amsdu_size`, `iwlwifi.swcrypto`, `iwlwifi.bt_coex_active`, `iwlwifi.power_level`, `iwlwifi.fw_restart`, `iwlwifi.bt_force_inactive`, `iwlwifi.fw_monitor`, `iwlwifi.no_bt_coex`, `iwlwifi.disable_11ac`, …
- `ath11k.frame_mode`, `ath11k.crypto_mode`, `ath11k.dump_qmi_history`, `ath11k.bdf_mac_ext`, …
- `mt76.dump_packet`, `mt7921.disable_aspm`, …
- `brcmfmac.feature_disable`, `brcmfmac.alternative_fw_path`, `brcmfmac.fcmode`, …

Wire-format identical (these param names appear in distro `/etc/modprobe.d/` files).

### Monitor mode + radiotap

Per-vif iftype `NL80211_IFTYPE_MONITOR`; injected pcap reads radiotap-headered frames. Behavior identical so `airmon-ng` / `wireshark monitor capture` / `tcpdump -i wlan0mon` work unchanged.

### AP mode + hostapd

`NL80211_IFTYPE_AP` + `NL80211_CMD_START_AP` with full BSS-config struct; consumed by hostapd. Identical wire format so hostapd config files apply unchanged.

### P2P / NaN

`NL80211_IFTYPE_P2P_CLIENT/_GO/_DEVICE` + NaN attrs identical; wpa_supplicant P2P implementation works unchanged.

### MLO + 802.11be Wi-Fi 7

`NL80211_CMD_*_MLO_LINK*` + per-link config; iwd + wpa_supplicant Wi-Fi 7 paths consume unchanged.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `net/wireless/cfg80211-core.md` | `core.c` + `nl80211.c` (chunks): cfg80211 wiphy + wdev lifecycle + nl80211 generic infra |
| `net/wireless/cfg80211-nl80211-cmds.md` | `nl80211.c` per-command handlers (split per-feature) |
| `net/wireless/cfg80211-wext.md` | `wext-*.c`: legacy Wireless Extensions ioctl |
| `net/wireless/cfg80211-scan.md` | `scan.c` + BSS database |
| `net/wireless/cfg80211-mlme.md` | `mlme.c`: cross-driver MLME helpers |
| `net/wireless/cfg80211-chan.md` | `chan.c` + `chan_trace.c`: channel context |
| `net/wireless/cfg80211-reg.md` | `reg.c` + `certs/`: regulatory database + signing key |
| `net/wireless/cfg80211-ibss-mesh-ocb.md` | `ibss.c` + `mesh.c` + `ocb.c`: ad-hoc/mesh/V2X modes |
| `net/wireless/cfg80211-ap.md` | `ap.c` |
| `net/wireless/cfg80211-pmsr.md` | `pmsr.c`: FTM 802.11mc |
| `net/wireless/cfg80211-radiotap.md` | `radiotap.c` |
| `net/wireless/cfg80211-debugfs-sysfs.md` | `debugfs.c` + `sysfs.c` |
| `net/wireless/mac80211-iface-cfg.md` | `iface.c` + `cfg.c` + `chan.c`: vif lifecycle + channel binding |
| `net/wireless/mac80211-mlme.md` | `mlme.c`: station MLME |
| `net/wireless/mac80211-rx.md` | `rx.c` |
| `net/wireless/mac80211-tx.md` | `tx.c` |
| `net/wireless/mac80211-sta-info.md` | `sta_info.c` |
| `net/wireless/mac80211-agg.md` | `agg-rx.c` + `agg-tx.c`: A-MPDU |
| `net/wireless/mac80211-key.md` | `key.c` + `aead_api.c` + `aes_*.c` + `michael-mic.c` |
| `net/wireless/mac80211-wpa.md` | `wpa.c` + `fils_aead.c` |
| `net/wireless/mac80211-mesh.md` | `mesh*.c`: HWMP + mesh peering + path table |
| `net/wireless/mac80211-ibss-ocb.md` | `ibss.c` + `ocb.c` |
| `net/wireless/mac80211-ht-vht-he-eht.md` | `ht.c` + `vht.c` + `he.c` + `eht.c` |
| `net/wireless/mac80211-mlo.md` | `mlo.c`: 802.11be MLO |
| `net/wireless/mac80211-rate-minstrel.md` | `rate.c` + `rc80211_minstrel_ht*.c` |
| `net/wireless/mac80211-airtime.md` | `airtime.c` |
| `net/wireless/mac80211-status.md` | `status.c` |
| `net/wireless/mac80211-debugfs.md` | `debugfs*.c` |
| `net/wireless/mac80211-vendor.md` | `vendor.c` |
| `net/wireless/iwlwifi.md` | `intel/iwlwifi/` (the dominant) |
| `net/wireless/ath9k.md` | `ath/ath9k/` |
| `net/wireless/ath10k.md` | `ath/ath10k/` |
| `net/wireless/ath11k.md` | `ath/ath11k/` |
| `net/wireless/ath12k.md` | `ath/ath12k/` (Wi-Fi 7) |
| `net/wireless/mt76.md` | `mediatek/mt76/` |
| `net/wireless/brcmfmac.md` | `broadcom/brcm80211/brcmfmac/` |
| `net/wireless/rtw88.md` | `realtek/rtw88/` |
| `net/wireless/rtw89.md` | `realtek/rtw89/` (Wi-Fi 7) |
| `net/wireless/rtl8xxxu.md` | `realtek/rtl8xxxu/` |
| `net/wireless/mwifiex.md` | `marvell/mwifiex/` |
| `net/wireless/rsi.md` | `rsi/` |
| `net/wireless/mac80211-hwsim.md` | `virtual/mac80211_hwsim.c`: software simulator (CI) |

### compatibility outline (top-level)

- REQ-O1: NETLINK_GENERIC `nl80211` family wire format byte-identical (wpa_supplicant + iwd + hostapd + iw + iwctl + NetworkManager + ConnMan consume unchanged).
- REQ-O2: Wireless Extensions legacy ioctl ABI byte-identical (iwconfig/iwlist work for legacy users).
- REQ-O3: Per-wiphy + per-vif + per-station sysfs + debugfs surface byte-identical (`iw phy/dev/info` consumes unchanged).
- REQ-O4: Regulatory database + per-country channel/power restrictions enforced identically (loaded from `/lib/firmware/regulatory.db` signed by Sforshee key).
- REQ-O5: Per-driver firmware load via `request_firmware` works for all v0 maintenance-set drivers.
- REQ-O6: Per-driver module params byte-identical (distro modprobe.d files apply).
- REQ-O7: Monitor mode + radiotap pcap capture byte-identical (airmon-ng + Wireshark monitor captures consume unchanged).
- REQ-O8: AP mode + hostapd integration: hostapd config files apply unchanged.
- REQ-O9: P2P / NaN / TDLS / FTM / 6 GHz / Wi-Fi 7 MLO + EHT features work identically on supporting HW.
- REQ-O10: Multiple-vif per-wiphy (concurrent STA + P2P + AP) selectable by NL80211_CMD_NEW_INTERFACE works.
- REQ-O11: Soft-MAC drivers (iwlwifi, ath9k/10k/11k/12k, mt76, rtw88/89, rtl8xxxu) AND full-MAC drivers (brcmfmac, mwifiex full-MAC mode) both interoperate via cfg80211 UAPI uniformly.
- REQ-O12: TLA+ models declared at this Tier-2 (mac80211 station MLME state machine, A-MPDU block-ack reorder buffer, regulatory state machine, mesh path-table HWMP convergence, MLO link-binding, key install/uninstall during 4-way-handshake).
- REQ-O13: Verus/Creusot Layer-4 functional contracts on cipher key install bounds, 802.11 IE parser bounds, BSS database refcount.
- REQ-O14: Hardening: row-1 features applied per `00-security-principles.md`, with wireless-specific reinforcement (regulatory DB signature verification, per-driver firmware signature verification, monitor-mode CAP_NET_ADMIN, KRACK-class-prevention key-reinstall protection, FT-OTA replay protection).

### acceptance criteria (top-level)

- [ ] AC-O1: `iw phy` + `iw dev wlan0 info` + `iw dev wlan0 scan` produce output byte-identical to upstream baseline. (covers REQ-O1, REQ-O3)
- [ ] AC-O2: wpa_supplicant connects to WPA2-PSK + WPA3-SAE + WPA3-SAE-Hash-to-Element networks; DHCP succeeds. (covers REQ-O1)
- [ ] AC-O3: iwd connects to same set of networks via its NL80211 path. (covers REQ-O1)
- [ ] AC-O4: hostapd brings up WPA2 + WPA3 AP; client connects + transfers data. (covers REQ-O8)
- [ ] AC-O5: `airmon-ng start wlan0` enters monitor mode; `tcpdump -i wlan0mon` captures 802.11 frames + radiotap headers. (covers REQ-O7)
- [ ] AC-O6: regulatory test: `iw reg set US` then `iw reg get` shows US regulatory; `/lib/firmware/regulatory.db` signature verified. (covers REQ-O4)
- [ ] AC-O7: 6 GHz test on Wi-Fi 6E HW: scan returns 6 GHz APs; LPI/SP transmit power gating works per-country regulatory. (covers REQ-O4, REQ-O9)
- [ ] AC-O8: 802.11be Wi-Fi 7 test on ath12k or rtw89: 320 MHz channel + EHT-MCS-15 + MLO 2-link aggregation works. (covers REQ-O9)
- [ ] AC-O9: P2P group-owner test: wpa_supplicant creates P2P GO; another device connects via P2P client; data transfer works. (covers REQ-O9)
- [ ] AC-O10: Multi-vif test: STA + P2P device + AP simultaneously on a single wiphy works. (covers REQ-O10)
- [ ] AC-O11: kselftest `tools/testing/selftests/net/wireless/` passes. (covers REQ-O12, REQ-O13)
- [ ] AC-O12: hwsim CI: 5-station mesh network via `mac80211_hwsim` converges + carries traffic. (covers REQ-O12)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/wireless/mlme_state.tla` | `net/wireless/mac80211-mlme.md` (proves: STA-mode MLME state machine — UNAUTHENTICATED → AUTHENTICATED → ASSOCIATED → AUTHORIZED (post-4-way-handshake); concurrent deauth + assoc-resp + key-install never produce stuck-state or stuck-with-installed-key-but-deauth'd; KRACK-class GTK-reinstall denied) |
| `models/wireless/ampdu_reorder.tla` | `net/wireless/mac80211-agg.md` (proves: A-MPDU reorder buffer per-tid — concurrent reception of out-of-order MPDUs + BA-window-shift never produces frame loss within window or duplicate delivery; gap timeout flushes correctly) |
| `models/wireless/regulatory_state.tla` | `net/wireless/cfg80211-reg.md` (proves: regulatory state machine — per-country regdom apply + per-driver hint accumulation + intersection produces consistent per-channel allow/deny mask; CRDA + kernel-DB conflict resolved by precedence; user-set country survives interface up/down) |
| `models/wireless/mesh_hwmp.tla` | `net/wireless/mac80211-mesh.md` (proves: HWMP routing in 802.11s mesh — per-mesh-station path-table convergence under N-station mesh; PREQ + PREP exchange produces shortest-airtime-metric path; concurrent peer disconnect + path-discovery never produce routing loop) |
| `models/wireless/mlo_link_binding.tla` | `net/wireless/mac80211-mlo.md` (proves: 802.11be MLO link binding — per-link state machine + per-link key install + per-link tid-to-link mapping; concurrent link-add/link-remove + tid-traffic never produce frame on inactive link or key-install on wrong link) |
| `models/wireless/key_install.tla` | `net/wireless/mac80211-key.md` (proves: per-link/per-station key install/uninstall + concurrent encrypt/decrypt — KRACK-class protection: GTK never reinstalled with same key; PTK rekey window correctly ordered with handshake completion) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `net/wireless/cfg80211-core.md` | wiphy refcount + wdev refcount: every CMD_GET_INTERFACE returns valid wdev with refcount-held; close + concurrent CMD_DEL_INTERFACE safe |
| `net/wireless/mac80211-key.md` | cipher key install bounds: per-cipher key length validated against expected (CCMP/GCMP=16 or 32, AES-CMAC=16, etc.); key-id within range; rx/tx PN counters monotonic |
| `net/wireless/cfg80211-scan.md` | BSS database invariant: per-BSS refcount; entry expiration timer fires exactly once; concurrent scan-result + scan-flush serialized |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-wiphy + per-wdev + per-bss + per-station + per-key + per-scan-request refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-driver `cfg80211_ops`, per-driver `ieee80211_ops`, per-driver `ieee80211_iface_combination` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | 802.11 IE length sums + frame-buffer arithmetic + per-station tid count multiplications use checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-driver cfg80211_ops + per-driver ieee80211_ops + per-cipher dispatch tables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed station + key state cleared (carries WPA PMK/PTK/GTK keys + 4-way-handshake transient state) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | wireless has no JIT — N/A | § Mandatory |

### Row-2 (LSM-stackable) features for wireless

LSM hooks called: file-LSM hooks on netlink + sysfs writes. `security_cfg80211_*` family (proposed for Rookery — upstream coverage minimal beyond CAP_NET_ADMIN check on each NL80211_CMD).

GR-RBAC adds:
- Per-role disallow `NL80211_IFTYPE_MONITOR` create (defends against unauthorized 802.11 capture).
- Per-role disallow regulatory-set CMD (defends against transmit-power escalation via fake-country regulatory).
- Per-role disallow vendor-private nl80211 commands (per-vendor commands have weakest validation).
- Per-role audit of every nl80211 CMD_AUTHENTICATE + CMD_ASSOCIATE + CMD_DEAUTHENTICATE (which uid + which BSS).

### Wireless-specific reinforcement

- **Regulatory DB signature verification**: `/lib/firmware/regulatory.db` MUST be loaded with valid Sforshee signature (built-in key under `net/wireless/certs/`); refuse to apply unsigned regulatory updates (defense against transmit-power escalation via fake regulatory DB).
- **Per-driver firmware signature verification**: per-driver firmware (iwlwifi, ath11k, mt7921, rtw89, brcmfmac, etc.) signature-verified before load via firmware-LSM hook (cross-ref `drivers/base/firmware-loader.md`).
- **Monitor mode requires CAP_NET_ADMIN**: NL80211_IFTYPE_MONITOR creation gated; tx-injection-mode further requires CAP_NET_RAW.
- **KRACK-class GTK reinstall protection**: mac80211 key-install logic refuses to reinstall a key with the same key-id + same key-data (CVE-2017-13077..13084 family); enforced in `mac80211/key.c`.
- **FT-OTA replay protection**: FT (Fast Transition) over-the-air re-association validates re-assoc seqnum > prior; replay rejected.
- **WPS PIN brute-force throttle**: wpa_supplicant-side mostly, but mac80211 deauth-flood detection rate-limits per-source MAC deauth handling.
- **6 GHz LPI/SP/VLP power gating**: Rookery enforces per-country LPI/SP/VLP power categories from regulatory DB; tx-power exceeding category rejected at mac80211 layer (defense against driver-bypassing regulatory limits).
- **MLO link-key isolation**: per-link keys never cross-installed; defense against link-bound attacks.
- **AP mode default WPA3-only**: hostapd-side, but Rookery's regulatory + cipher-suite advertisement default to WPA3 (SAE) + 802.11w (PMF) required for new APs; legacy WPA2 explicit-opt-in.
- **TKIP cipher disabled by default**: TKIP is broken; Rookery's mac80211 cipher allowlist rejects TKIP for new associations unless CONFIG_MAC80211_RC4=y + explicit module param `mac80211.allow_tkip=1` (legacy compat only).
- **Vendor private nl80211 commands rate-limited + audited**: per-vendor commands have weakest validation surface; rate-limit per-fd + audit-log.

