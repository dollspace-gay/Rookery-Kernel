# Tier-3: net/mac80211/main.c — mac80211 softMAC core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/wireless/00-overview.md
upstream-paths:
  - net/mac80211/main.c (~1832 lines)
  - net/mac80211/ieee80211_i.h (struct ieee80211_local, sub_if_data)
  - net/mac80211/rx.c (RX handler chain)
  - net/mac80211/tx.c (TX handler chain)
  - net/mac80211/sta_info.{c,h} (struct sta_info)
  - net/mac80211/key.{c,h} (struct ieee80211_key)
  - include/net/mac80211.h (driver-facing API)
-->

## Summary

`mac80211` is the Linux **softMAC** IEEE-802.11 stack. Hardware drivers that do not implement full-MAC in firmware register a `struct ieee80211_hw` with mac80211, which then runs MLME (association, authentication, scanning), QoS scheduling, fragmentation, sequence-numbering, encryption (WEP/TKIP/CCMP/GCMP/BIP), aggregation (A-MPDU / A-MSDU), power-save, and per-AC queueing in software. Per-radio state lives in `struct ieee80211_local` (allocated as `wiphy_priv(wiphy)` and accessible to drivers via `hw->priv`); per-virtual-interface state in `struct ieee80211_sub_if_data`; per-peer state in `struct sta_info`; per-key state in `struct ieee80211_key`. Per-RX path is a deterministic chain of `ieee80211_rx_h_*` handlers driven by `CALL_RXH(rxh)` returning `RX_CONTINUE` / `RX_DROP*` / `RX_QUEUED`; per-TX path is the dual chain `ieee80211_tx_h_*`. Per-tasklet (`local->tasklet`, `ieee80211_tasklet_handler`) drains `local->skb_queue` and `skb_queue_unreliable`. Per-AC there are four queues (`IEEE80211_NUM_ACS`) plus pending queues `local->pending[IEEE80211_MAX_QUEUES]`. `main.c` itself is the hw lifetime / configuration center: `ieee80211_alloc_hw_nm`, `ieee80211_register_hw`, `ieee80211_unregister_hw`, `ieee80211_free_hw`, `ieee80211_restart_hw`, plus filter / chanctx / band-cap plumbing. Critical for: surviving all in-tree 802.11 drivers (iwlwifi, ath9k/ath10k/ath11k/ath12k, mt76, rt2x00, mwifiex, mac80211_hwsim) and userspace nl80211 / wpa_supplicant / hostapd.

This Tier-3 covers `net/mac80211/main.c` (~1832 lines), the lifetime / config core of the mac80211 module. RX-handler-chain internals, TX-handler-chain internals, MLME, scanning, rate control, sta_info lifetime, and key management each have their own Tier-3 docs; this doc references them as the central wiring file.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ieee80211_local` | per-radio softMAC state | `Mac80211Local` |
| `struct ieee80211_hw` | per-driver public handle | `Ieee80211Hw` |
| `struct ieee80211_ops` | per-driver callback vtable | `Ieee80211Ops` (trait) |
| `struct ieee80211_sub_if_data` | per-vif softMAC state | `SubIfData` |
| `struct sta_info` | per-peer state | `StaInfo` |
| `struct ieee80211_key` | per-key state | `Ieee80211Key` |
| `ieee80211_alloc_hw_nm()` | per-allocation w/ name | `Mac80211::alloc_hw_nm` |
| `ieee80211_register_hw()` | per-registration | `Mac80211::register_hw` |
| `ieee80211_unregister_hw()` | per-teardown | `Mac80211::unregister_hw` |
| `ieee80211_free_hw()` | per-free | `Mac80211::free_hw` |
| `ieee80211_restart_hw()` | per-driver-reset | `Mac80211::restart_hw` |
| `ieee80211_restart_work()` | per-restart workqueue | `Mac80211::restart_work` |
| `ieee80211_tasklet_handler()` | per-radio bottom-half | `Mac80211::tasklet_handler` |
| `ieee80211_handle_queued_frames()` | per-tasklet drain | `Mac80211::handle_queued_frames` |
| `ieee80211_configure_filter()` | per-mc / per-promisc filter | `Mac80211::configure_filter` |
| `ieee80211_reconfig_filter()` | per-wiphy_work | `Mac80211::reconfig_filter` |
| `ieee80211_hw_config()` | per-radio HW conf push | `Mac80211::hw_config` |
| `ieee80211_hw_conf_chan()` | per-chanctx conf push | `Mac80211::hw_conf_chan` |
| `ieee80211_calc_hw_conf_chan()` | per-chandef compute | `Mac80211::calc_hw_conf_chan` |
| `ieee80211_bss_info_change_notify()` | per-BSS change → driver | `Mac80211::bss_info_change_notify` |
| `ieee80211_link_info_change_notify()` | per-MLO link change | `Mac80211::link_info_change_notify` |
| `ieee80211_vif_cfg_change_notify()` | per-vif cfg change | `Mac80211::vif_cfg_change_notify` |
| `ieee80211_init_cipher_suites()` | per-radio cipher init | `Mac80211::init_cipher_suites` |
| `ieee80211_emulate_{add,remove,change}_chanctx` | per-chanctx emul | `Mac80211::emulate_chanctx_*` |
| `ieee80211_ifa_changed()` / `ifa6_changed()` | per-inet(6)addr notifier | `Mac80211::ifa_changed` / `ifa6_changed` |
| `ieee80211_ifcomb_check()` | per-iface-combination validation | `Mac80211::ifcomb_check` |
| `ieee80211_create_default_chandef()` | per-band default chan | `Mac80211::create_default_chandef` |
| `ieee80211_init()` / `ieee80211_exit()` | per-module init/exit | `Mac80211::module_init` / `_exit` |
| `panic_notifier_list` (no — separate file) | n/a | n/a |
| RX chain `CALL_RXH(ieee80211_rx_h_*)` | per-skb handler pipeline | `Mac80211::rx_h_chain` (in `rx.md`) |
| TX chain `CALL_TXH(ieee80211_tx_h_*)` | per-skb handler pipeline | `Mac80211::tx_h_chain` (in `tx.md`) |

## Compatibility contract

REQ-1: struct ieee80211_local (subset relevant to main.c):
- hw: embedded `struct ieee80211_hw` (driver-facing; `wiphy`, `priv`, `conf`, `queues`, `max_rates`, `extra_tx_headroom`, `flags`, `radiotap_*`, `uapsd_*`, `max_mtu`, `max_listen_interval`, `weight_multiplier`, `max_nan_de_entries`).
- ops: `const struct ieee80211_ops *` (driver callbacks).
- emulate_chanctx: bool — per-driver-uses-emulated-chanctx flag.
- monitors / fif_* / probe_req_reg / rx_mcast_action_reg: per-filter shadow counters.
- filter_flags: per-current FIF_ mask pushed to driver.
- filter_lock: spinlock — per-filter shadow + drv_prepare_multicast.
- rx_path_lock: spinlock — per-rx serial section.
- queue_stop_reason_lock: spinlock — per-queue-stop reason bitmap.
- handle_wake_tx_queue_lock: spinlock — per-driver wake-tx-queue entry.
- iflist_mtx: mutex — per-interfaces list mutator.
- interfaces: list of `ieee80211_sub_if_data`.
- mon_list: list of monitor sdata.
- mc_list: `struct netdev_hw_addr_list` per-radio multicast list.
- ack_status_lock + ack_status_frames (idr): per-ack-tracking.
- skb_queue / skb_queue_unreliable: per-tasklet RX + TX-status input.
- pending[IEEE80211_MAX_QUEUES]: per-AC pending skb queue (post-stop reroute).
- active_txqs[IEEE80211_NUM_ACS] / active_txq_lock[IEEE80211_NUM_ACS]: per-AC airtime-fairness scheduler queues.
- aql_*: per-airtime-queue-limit accounting.
- airtime_flags: AIRTIME_USE_TX | AIRTIME_USE_RX.
- tasklet / tx_pending_tasklet / wake_txqs_tasklet: bottom-half tasklets.
- workqueue: ordered workqueue (created in register_hw).
- restart_work: per-driver-restart work_struct.
- reconfig_filter / radar_detected_work / dynamic_ps_{enable,disable}_work / sched_scan_stopped_work: per-wiphy_work entries.
- dynamic_ps_timer: per-PS-state timer.
- scan_work / scanning bitmap / scan_chandef / tmp_channel / scan_ies_len / int_scan_req: per-scan state.
- chanctx_list: list of channel contexts.
- monitor_chanreq / dflt_chandef: per-monitor / per-default channel.
- ifa_notifier / ifa6_notifier: per-IPv4/IPv6 addr-change notifier blocks.
- rate_ctrl: rate control algorithm handle.
- rx_chains: per-radio rx chain count.
- sband_allocated: bitmap of sbands mac80211 reallocated (for VHT EXT NSS BW).
- ext_capa[]: per-radio HT/VHT extended cap bytes.
- agg_queue_stop[IEEE80211_MAX_QUEUES] (atomic): per-AC aggregation stop counter.

REQ-2: struct ieee80211_hw (public driver handle):
- wiphy: cfg80211 wiphy pointer (1:1 with radio).
- priv: driver private buffer (after `local`, NETDEV_ALIGN'd).
- conf: `struct ieee80211_conf` (chandef, power_level, smps_mode, listen_interval, long/short retry).
- flags: per-cap bitmap (`IEEE80211_HW_*`).
- queues: per-AC HW queue count (≥ IEEE80211_NUM_ACS for HT/VHT/HE).
- offchannel_tx_hw_queue: per-off-channel TX HW queue index.
- max_rates / max_report_rates / max_rx_aggregation_subframes / max_tx_aggregation_subframes.
- extra_tx_headroom: per-driver TX skb headroom requirement.
- radiotap_*: per-radiotap injection fields.
- uapsd_queues / uapsd_max_sp_len: per-U-APSD config.
- max_mtu: per-MTU cap.
- tx_sk_pacing_shift: TCP small-queues pacing shift (default 7).
- max_listen_interval: per-listen-interval cap.
- max_nan_de_entries: per-NAN DE entry cap.
- weight_multiplier: per-airtime weight.

REQ-3: struct ieee80211_ops (driver vtable; mandatory subset enforced):
- tx, start, stop, config, add_interface, remove_interface, configure_filter, wake_tx_queue — must be non-NULL or `ieee80211_alloc_hw_nm` returns NULL.
- sta_state XOR (sta_add | sta_remove): must not coexist.
- link_info_changed iff vif_cfg_changed (MLO-aware drivers).
- bss_info_changed mutually exclusive with link_info_changed.
- chanctx callbacks: either all emulated (add/remove/change == `ieee80211_emulate_*`) and assign/unassign_vif_chanctx NULL; or all real (add/remove/change/assign/unassign non-emulated and non-NULL).

REQ-4: ieee80211_alloc_hw_nm(priv_data_len, ops, requested_name):
- /* Validate ops */ — per REQ-3.
- /* Layout: wiphy | ieee80211_local (NETDEV_ALIGN) | driver-priv */
- priv_size = ALIGN(sizeof(*local), NETDEV_ALIGN) + priv_data_len.
- wiphy = wiphy_new_nm(&mac80211_config_ops, priv_size, requested_name).
- if !wiphy: return NULL.
- /* wiphy defaults */ — mgmt_stypes, privid, flags (NETNS_OK | 4ADDR_AP | 4ADDR_STATION | REPORTS_OBSS | OFFCHAN_TX), bss_param_support, features (SK_TX_STATUS | SAE | HT_IBSS | VIF_TXPOWER | MAC_ON_CREATE | USERSPACE_MPM | FULL_AP_CLIENT_STATE), ext_feature (FILS_STA, CONTROL_PORT_*, SCAN_FREQ_KHZ, POWERED_ADDR_CHANGE, TXQS, RRM, IEEE8021X_AUTH).
- /* Has-remain-on-channel: emulate or driver-provided */
- /* if hw_scan absent: add LOW_PRIORITY_SCAN | AP_SCAN + SCAN_RANDOM_SN + SCAN_MIN_PREQ_CONTENT */.
- /* if set_key absent (SW-crypto-only): WIPHY_FLAG_IBSS_RSN + SPP_AMSDU_SUPPORT */.
- wiphy->bss_priv_size = sizeof(struct ieee80211_bss).
- local = wiphy_priv(wiphy).
- /* sta_info_init: may fail → wiphy_free → NULL */
- local->hw.wiphy = wiphy.
- local->hw.priv = (char *)local + ALIGN(sizeof(*local), NETDEV_ALIGN).
- local->ops = ops; local->emulate_chanctx = emulate_chanctx.
- /* Defaults: queues=1, max_rates=1, max_rx/tx_aggregation_subframes = IEEE80211_MAX_AMPDU_BUF_HT, offchannel_tx_hw_queue = IEEE80211_INVAL_HW_QUEUE, retry counts from wiphy, uapsd defaults, max_mtu=IEEE80211_MAX_DATA_LEN */.
- /* Lists / locks init: interfaces, mon_list, mc_list, iflist_mtx, filter_lock, rx_path_lock, queue_stop_reason_lock, handle_wake_tx_queue_lock, chanctx_list, active_txqs[]/active_txq_lock[] per AC, aql_*, ack_status_lock + idr_init(ack_status_frames), pending[] skb queues, agg_queue_stop[] */.
- /* Work items: scan_work, restart_work, radar_detected_work, reconfig_filter, dynamic_ps_{enable,disable}_work, dynamic_ps_timer, sched_scan_stopped_work */.
- /* Tasklets: tx_pending_tasklet → ieee80211_tx_pending, wake_txqs_tasklet → ieee80211_wake_txqs, tasklet → ieee80211_tasklet_handler */.
- skb_queue_head_init(skb_queue, skb_queue_unreliable).
- ieee80211_alloc_led_names(local).
- ieee80211_roc_setup(local).
- return &local->hw.

REQ-5: ieee80211_register_hw(hw):
- /* Per-cap consistency: QUEUE_CONTROL requires valid offchannel_tx_hw_queue */
- /* TDLS_CHANNEL_SWITCH requires tdls_channel_switch / cancel / recv callbacks */
- /* SUPPORTS_TX_FRAG requires set_frag_threshold */
- /* NAN iface mode requires start_nan + stop_nan */
- /* MLO (WIPHY_FLAG_SUPPORTS_MLO) requires: !emulate_chanctx, link_info_changed, HAS_RATE_CONTROL, AMPDU_AGGREGATION, !HOST_BROADCAST_PS_BUFFERING, (PS ⇒ SUPPORTS_DYNAMIC_PS ∧ !PS_NULLFUNC_STACK), MFP_CAPABLE, !NEED_DTIM_BEFORE_ASSOC, !TIMING_BEACON_ONLY, AP_LINK_PS */
- /* WoWLAN requires suspend + resume */
- /* If emulate_chanctx: every iface_combination must have num_different_channels ≤ 1 */
- /* Per-radio (n_radio > 0): ieee80211_ifcomb_check each radio's iface_combinations; else ifcomb_check the wiphy-level one */
- /* netdev_features must be subset of MAC80211_SUPPORTED_FEATURES */
- if hw->max_report_rates == 0: max_report_rates = max_rates.
- local->rx_chains = 1.
- /* Per-band scan: pick default channel (skip CHAN_DISABLED | S1G_NO_PRIMARY), compute supp_{ht,vht,he,eht,s1g,uhr}, max_bitrates, channels++. For each iftd: HE-40MHz consistency check; vendor_elems must be empty under MLO */
- /* HT/VHT/HE require ≥ IEEE80211_NUM_ACS queues; EHT requires HE; UHR requires EHT */
- /* If HT: SM_PS_DISABLED bits set */
- /* If AP supported AND !SW_CRYPTO_CONTROL: add AP_VLAN to interface_modes + software_iftypes */
- /* Monitor always supported: add NL80211_IFTYPE_MONITOR to interface_modes + software_iftypes */
- /* int_scan_req allocation */
- /* !CONFIG_MAC80211_MESH: remove MESH_POINT */
- /* Mesh → WIPHY_FLAG_MESH_AUTH */
- /* WIPHY_FLAG_CONTROL_PORT_PROTOCOL set unconditionally */
- /* Signal type: SIGNAL_DBM → CFG80211_SIGNAL_TYPE_MBM; SIGNAL_UNSPEC → CFG80211_SIGNAL_TYPE_UNSPEC (max_signal > 0 required) */
- /* SW-crypto-only: CAN_REPLACE_PTK0, EXT_KEY_ID */
- /* IBSS → DEL_IBSS_STA ext feature */
- /* scan_ies_len computed from supp_{ht,vht,s1g,he,eht,uhr} cap sizes */
- /* if !hw_scan: max_scan_ssids=4, max_scan_ie_len=IEEE80211_MAX_DATA_LEN */
- /* Subtract local->scan_ies_len from wiphy->max_scan_ie_len */
- result = ieee80211_init_cipher_suites(local). On failure: goto fail_workqueue.
- /* if !remain_on_channel: max_remain_on_channel_duration = 5000 ms */
- /* SUPPORTS_TDLS → TDLS_EXTERNAL_SETUP */
- /* CHANCTX_STA_CSA → WLAN_EXT_CAPA1_EXT_CHANNEL_SWITCHING */
- /* SUPPORTS_MULTI_BSSID: support_mbssid, optional only_he_mbssid, EXT_CAPA3_MULTI_BSSID_SUPPORT */
- /* max_num_csa_counters = IEEE80211_MAX_CNTDWN_COUNTERS_NUM */
- /* queues clamped to IEEE80211_MAX_QUEUES */
- /* workqueue = alloc_ordered_workqueue (failure → fail_workqueue) */
- /* tx_headroom = max(extra_tx_headroom, IEEE80211_TX_STATUS_HEADROOM) */
- /* max_listen_interval default 5; conf.listen_interval set; dynamic_ps_forced_timeout = -1; max_nan_de_entries default IEEE80211_MAX_NAN_INSTANCE_ID; weight_multiplier default 1 */
- ieee80211_wep_init(local).
- local->hw.conf.flags = IEEE80211_CONF_IDLE.
- ieee80211_led_init(local).
- result = ieee80211_txq_setup_flows(local). On failure: fail_flows.
- /* rtnl_lock(); ieee80211_init_rate_ctrl_alg(local, hw->rate_control_algorithm); rtnl_unlock() */ — failure → fail_rate.
- /* rate_ctrl ⇒ VHT_EXT_NSS_BW cap (per RATE_CTRL_CAPA_VHT_EXT_NSS_BW) */
- /* Per-band VHT EXT NSS BW: if mismatched, kmemdup sband, toggle the bit, record in sband_allocated */
- /* ASSOC_FRAME_ENCRYPTION → EPPKE */
- result = wiphy_register(local->hw.wiphy). On failure: fail_wiphy_register.
- debugfs_hw_add(local); rate_control_add_debugfs(local).
- ieee80211_check_wbrf_support(local).
- /* rtnl_lock + wiphy_lock: add default wlan%d STA iface if STATION supported and !NO_AUTO_VIF */
- /* Register inet / inet6 addr notifiers (ifa_changed / ifa6_changed) */
- return 0.
- Failure ladder: fail_ifa6 → unreg ifa → fail_ifa → wiphy_unregister → fail_wiphy_register → rtnl_lock + rate_control_deinitialize + ieee80211_remove_interfaces → fail_rate → ieee80211_txq_teardown_flows → fail_flows → ieee80211_led_exit + destroy_workqueue → fail_workqueue → kfree(int_scan_req) → return result.

REQ-6: ieee80211_unregister_hw(hw):
- tasklet_kill(tx_pending_tasklet); tasklet_kill(tasklet).
- unregister_inetaddr_notifier(ifa_notifier); unregister_inet6addr_notifier(ifa6_notifier).
- rtnl_lock. ieee80211_remove_interfaces(local). ieee80211_txq_teardown_flows(local).
- wiphy_lock: cancel delayed roc_work, reconfig_filter, sched_scan_stopped_work, radar_detected_work. wiphy_unlock. rtnl_unlock.
- cancel_work_sync(restart_work).
- ieee80211_clear_tx_pending(local).
- rate_control_deinitialize(local).
- /* skb_queue / skb_queue_unreliable must be empty — WARN otherwise */
- skb_queue_purge both.
- wiphy_unregister(wiphy). destroy_workqueue(workqueue). ieee80211_led_exit. kfree(int_scan_req).

REQ-7: ieee80211_free_hw(hw):
- mutex_destroy(iflist_mtx).
- idr_for_each(ack_status_frames, ieee80211_free_ack_frame) — WARN_ONCE per pending ack frame, kfree_skb.
- idr_destroy(ack_status_frames).
- sta_info_stop(local).
- ieee80211_free_led_names(local).
- /* Free per-band sbands that mac80211 reallocated (sband_allocated bit set) */
- wiphy_free(wiphy).

REQ-8: ieee80211_restart_hw(hw):
- trace_api_restart_hw.
- wiphy_info("Hardware restart was requested").
- /* Stop all queues with reason QUEUE_STOP_REASON_SUSPEND (unblocked by reconfig) */
- local->in_reconfig = true; barrier().
- queue_work(system_freezable_wq, &local->restart_work).

REQ-9: ieee80211_restart_work(work):
- flush_workqueue(local->workqueue).
- rtnl_lock; wiphy_lock; wiphy_work_flush(NULL).
- WARN if SCAN_HW_SCANNING.
- For each sdata in interfaces: STATION → wiphy_work_cancel(csa_connection_drop_work); if csa_active: ieee80211_sta_connection_lost(WLAN_REASON_UNSPECIFIED). Flush dec_tailroom_needed_wk.
- ieee80211_scan_cancel(local).
- wiphy_delayed_work_flush(roc_work); wiphy_work_flush(hw_roc_done).
- synchronize_net.
- ret = ieee80211_reconfig(local).
- wiphy_unlock.
- if ret: cfg80211_shutdown_all_interfaces(wiphy).
- rtnl_unlock.

REQ-10: ieee80211_tasklet_handler(t):
- local = from_tasklet(local, t, tasklet).
- ieee80211_handle_queued_frames(local).

REQ-11: ieee80211_handle_queued_frames(local):
- while ((skb = skb_dequeue(skb_queue)) || (skb = skb_dequeue(skb_queue_unreliable))):
  - switch pkt_type:
    - IEEE80211_RX_MSG → skb->pkt_type=0; ieee80211_rx(&hw, skb).
    - IEEE80211_TX_STATUS_MSG → skb->pkt_type=0; ieee80211_tx_status_skb(&hw, skb).
    - default → WARN("unknown type %d"); dev_kfree_skb.

REQ-12: ieee80211_configure_filter(local):
- new_flags = 0.
- if atomic_read(iff_allmultis) > 0: new_flags |= FIF_ALLMULTI.
- if monitors > 0 ∨ SCAN_SW_SCANNING ∨ SCAN_ONCHANNEL_SCANNING: new_flags |= FIF_BCN_PRBRESP_PROMISC.
- if fif_probe_req ∨ probe_req_reg: FIF_PROBE_REQ.
- if fif_fcsfail: FIF_FCSFAIL.
- if fif_plcpfail: FIF_PLCPFAIL.
- if fif_control: FIF_CONTROL.
- if fif_other_bss: FIF_OTHER_BSS.
- if fif_pspoll: FIF_PSPOLL.
- if rx_mcast_action_reg: FIF_MCAST_ACTION.
- spin_lock_bh(filter_lock). changed_flags = filter_flags ^ new_flags. mc = drv_prepare_multicast(local, &mc_list). spin_unlock_bh.
- /* Sentinel bit-31 must be cleared by driver — WARN otherwise */
- new_flags |= (1<<31).
- drv_configure_filter(local, changed_flags, &new_flags, mc).
- WARN_ON(new_flags & (1<<31)).
- filter_flags = new_flags & ~(1<<31).

REQ-13: RX-handler chain (`rx.md` detail; referenced here):
- Per-skb sk_buff arrives via `ieee80211_rx_napi` / `ieee80211_rx_irqsafe` / `ieee80211_rx_list` (`rx.c`).
- Per-frame builds `struct ieee80211_rx_data` (local, sdata, sta, key, flags, security_idx, seqno_idx).
- `CALL_RXH(rxh)` macro: `res = rxh(rx); if (res != RX_CONTINUE) goto rxh_next;`.
- Pre-PS / pre-association chain: `ieee80211_rx_h_check_dup`, `ieee80211_rx_h_check`.
- Main chain: `ieee80211_rx_h_check_more_data`, `_uapsd_and_pspoll`, `_sta_process`, `_decrypt`, `_defragment`, `_michael_mic_verify`, `_amsdu`, `_data`, `_ctrl`, `_mgmt_check`, `_action`, `_userspace_mgmt`, `_action_post_userspace`, `_action_return`, `_ext`, `_mgmt`.
- Return values: `RX_CONTINUE` (next handler), `RX_DROP_UNUSABLE` / `RX_DROP_U_*` (free + drop_reason), `RX_DROP_MONITOR` (free silently / pass to monitor), `RX_QUEUED` (handler owns skb).
- Drop reasons registered as `mac80211_unusable` subsys via `drops_reason_register_subsys` in `ieee80211_init`.

REQ-14: TX-handler chain (`tx.md` detail; referenced here):
- Per-skb enters via `ieee80211_subif_start_xmit` (netdev xmit) or `ieee80211_tx_skb` (mgmt).
- `struct ieee80211_tx_data` (local, sdata, sta, key, skb, skbs, flags).
- `CALL_TXH(txh)`: identical macro semantics to RX.
- Chain: `ieee80211_tx_h_dynamic_ps`, `_check_assoc`, `_ps_buf`, `_check_control_port_protocol`, `_select_key`, `_rate_ctrl` (optional), `_michael_mic_add`, `_sequence`, `_fragment`, `_stats`, `_encrypt`, `_calculate_duration`.
- After chain: per-AC enqueue via `ieee80211_drv_tx` (driver `wake_tx_queue` or legacy `tx`).
- Per-AC airtime fairness: `local->active_txqs[ac]`, `local->aql_ac_pending_airtime[ac]`, `local->aql_txq_limit_{low,high}[ac]`, `local->airtime_flags`.

REQ-15: Cipher-suite negotiation (ieee80211_init_cipher_suites):
- Default list: WEP40, WEP104, TKIP, CCMP, CCMP_256, GCMP, GCMP_256, AES_CMAC, BIP_CMAC_256, BIP_GMAC_128, BIP_GMAC_256.
- SW_CRYPTO_CONTROL + fips_enabled → -EINVAL (driver must offer its own).
- SW_CRYPTO_CONTROL ∧ !wiphy->cipher_suites → -EINVAL.
- fips_enabled ∨ !cipher_suites: assign default list. n = ARRAY_SIZE.
- !MFP_CAPABLE: n -= 4 (strip MFP ciphers at tail).
- fips_enabled: skip first 3 (RC4-based: WEP40, WEP104, TKIP).

REQ-16: Notifier integration:
- ieee80211_ifa_changed (IPv4): per-`ifa->ifa_dev->dev` lookup → wdev → sdata. STATION only. ARP-filter (`arp_addr_list[IEEE80211_BSS_ARP_ADDR_LIST_LEN]`). If `mgd.associated`: `ieee80211_vif_cfg_change_notify(sdata, BSS_CHANGED_ARP_FILTER)`.
- ieee80211_ifa6_changed (IPv6): STATION only. `drv_ipv6_addr_change`.

REQ-17: Module init / exit:
- ieee80211_init (subsys_initcall):
  - BUILD_BUG_ON(sizeof(struct ieee80211_tx_info) > sizeof(skb->cb)).
  - BUILD_BUG_ON(offsetof(driver_data) + IEEE80211_TX_INFO_DRIVER_DATA_SIZE > sizeof(skb->cb)).
  - rc80211_minstrel_init → ieee80211_iface_init → drop_reasons_register_subsys(SKB_DROP_REASON_SUBSYS_MAC80211_UNUSABLE).
- ieee80211_exit (module_exit):
  - rc80211_minstrel_exit → ieee80211s_stop → ieee80211_iface_exit → drop_reasons_unregister_subsys → rcu_barrier.

REQ-18: Per-key (struct ieee80211_key) — referenced (`key.md` detail):
- Fields: local, sdata, sta, list, debugfs entry, conf (driver-visible), flags, hw_key_idx, color, cipher type (TKIP/CCMP/GCMP/CMAC/...).
- Lifecycle: `ieee80211_key_alloc`, `ieee80211_key_link` (replaces existing PTK0 per `CAN_REPLACE_PTK0`), `ieee80211_key_free`.
- Used by: rx-h-decrypt, tx-h-select-key, tx-h-encrypt, tx-h-michael-mic-add, rx-h-michael-mic-verify.

REQ-19: Per-station (struct sta_info) — referenced (`sta_info.md` detail):
- Per-peer state: addr, sta (driver-visible `ieee80211_sta`), local, sdata, key (PTK), gtk[NUM_DEFAULT_KEYS], aid, ampdu_mlme, rx_stats, tx_stats, ps_lock, ps_tx_buf, tx_filtered, listen_interval.
- Initialized in `ieee80211_alloc_hw_nm` via `sta_info_init(local)`; torn down in `ieee80211_free_hw` via `sta_info_stop(local)`.

## Acceptance Criteria

- [ ] AC-1: ieee80211_alloc_hw_nm with NULL mandatory op (tx / start / stop / config / add_interface / remove_interface / configure_filter / wake_tx_queue) returns NULL.
- [ ] AC-2: ieee80211_alloc_hw_nm with mixed emulate-and-real chanctx ops returns NULL.
- [ ] AC-3: hw->priv 32-byte aligned past sizeof(*local).
- [ ] AC-4: ieee80211_register_hw with MLO + emulate_chanctx → -EINVAL.
- [ ] AC-5: ieee80211_register_hw with HT-supported sband but queues < IEEE80211_NUM_ACS → -EINVAL.
- [ ] AC-6: ieee80211_register_hw success: NL80211_IFTYPE_MONITOR present in interface_modes; AP_VLAN added if AP supported and !SW_CRYPTO_CONTROL.
- [ ] AC-7: ieee80211_register_hw with NO_AUTO_VIF set: no default wlan%d STA interface created.
- [ ] AC-8: ieee80211_unregister_hw kills tasklet + tx_pending_tasklet before iflist teardown.
- [ ] AC-9: ieee80211_free_hw: idr_destroy(ack_status_frames) and sta_info_stop run; warns if any pending ack frames.
- [ ] AC-10: ieee80211_restart_hw: queues stopped with reason SUSPEND, in_reconfig=true, restart_work queued on system_freezable_wq.
- [ ] AC-11: ieee80211_handle_queued_frames: IEEE80211_RX_MSG dispatched to ieee80211_rx; IEEE80211_TX_STATUS_MSG to ieee80211_tx_status_skb; unknown WARNed.
- [ ] AC-12: ieee80211_configure_filter: bit-31 sentinel cleared by driver (WARN otherwise); filter_flags reflects driver-accepted mask.
- [ ] AC-13: ieee80211_init_cipher_suites: SW_CRYPTO_CONTROL + fips_enabled → -EINVAL; !MFP_CAPABLE strips last 4 ciphers; fips_enabled strips first 3 (WEP+TKIP).
- [ ] AC-14: ieee80211_ifa_changed for non-STATION sdata → NOTIFY_DONE.
- [ ] AC-15: BUILD_BUG_ON in module init: sizeof(ieee80211_tx_info) ≤ sizeof(skb->cb).

## Architecture

```
struct Mac80211Local {
  hw: Ieee80211Hw,                       // embedded; driver-visible
  ops: &'static Ieee80211Ops,
  emulate_chanctx: bool,

  // Filtering
  filter_lock: SpinLock,
  filter_flags: u32,                     // FIF_*
  monitors: u32,
  fif_probe_req: u32,
  probe_req_reg: u32,
  fif_fcsfail: u32, fif_plcpfail: u32, fif_control: u32,
  fif_other_bss: u32, fif_pspoll: u32, rx_mcast_action_reg: u32,
  iff_allmultis: AtomicU32,
  mc_list: NetdevHwAddrList,

  // Locking
  rx_path_lock: SpinLock,
  queue_stop_reason_lock: SpinLock,
  handle_wake_tx_queue_lock: SpinLock,
  iflist_mtx: Mutex,
  ack_status_lock: SpinLock,
  ack_status_frames: Idr<SkBuff>,

  // Interfaces / vifs
  interfaces: ListHead<SubIfData>,
  mon_list: ListHead<SubIfData>,
  chanctx_list: ListHead<ChanCtx>,
  monitor_chanreq: ChanReq,
  dflt_chandef: ChanDef,

  // TX / queues
  pending: [SkbQueue; IEEE80211_MAX_QUEUES],
  active_txqs: [ListHead<Txq>; IEEE80211_NUM_ACS],
  active_txq_lock: [SpinLock; IEEE80211_NUM_ACS],
  aql_txq_limit_low: [u32; IEEE80211_NUM_ACS],
  aql_txq_limit_high: [u32; IEEE80211_NUM_ACS],
  aql_ac_pending_airtime: [AtomicU32; IEEE80211_NUM_ACS],
  aql_total_pending_airtime: AtomicU32,
  aql_threshold: u32,
  airtime_flags: u32,                    // AIRTIME_USE_TX|RX
  agg_queue_stop: [AtomicU32; IEEE80211_MAX_QUEUES],
  tx_headroom: u32,

  // Bottom halves
  tasklet: Tasklet,                      // ieee80211_tasklet_handler
  tx_pending_tasklet: Tasklet,
  wake_txqs_tasklet: Tasklet,
  skb_queue: SkbQueue,
  skb_queue_unreliable: SkbQueue,

  // Workqueues
  workqueue: OrderedWorkqueue,
  restart_work: Work,
  reconfig_filter: WiphyWork,
  radar_detected_work: WiphyWork,
  sched_scan_stopped_work: WiphyWork,
  dynamic_ps_enable_work: WiphyWork,
  dynamic_ps_disable_work: WiphyWork,
  dynamic_ps_timer: Timer,
  scan_work: WiphyDelayedWork,

  // Scanning
  scanning: AtomicUsize,                 // bitmap of SCAN_* flags
  scan_chandef: ChanDef,
  tmp_channel: Option<*IeeeChannel>,
  int_scan_req: *ScanRequest,
  scan_ies_len: u32,

  // Per-band scratchspace
  rx_chains: u32,
  sband_allocated: u8,                   // bitmap NUM_NL80211_BANDS
  ext_capa: [u8; WLAN_EXT_CAPA_LEN],

  // Rate control
  rate_ctrl: Option<RateControlRef>,

  // Notifiers
  ifa_notifier: NotifierBlock,           // CONFIG_INET
  ifa6_notifier: NotifierBlock,          // CONFIG_IPV6

  // Power / config state
  in_reconfig: bool,
  user_power_level: i32,
  dynamic_ps_forced_timeout: i32,
}

struct Ieee80211Hw {                     // public driver handle
  wiphy: *Wiphy,
  priv: *mut u8,                         // (char *)local + ALIGN(sizeof(*local), NETDEV_ALIGN)
  conf: Ieee80211Conf,
  flags: BitMap<IEEE80211_HW_LAST>,
  queues: u32,
  offchannel_tx_hw_queue: u16,
  max_rates: u8,
  max_report_rates: u8,
  max_rx_aggregation_subframes: u8,
  max_tx_aggregation_subframes: u8,
  extra_tx_headroom: u32,
  radiotap_mcs_details: u8,
  radiotap_vht_details: u8,
  radiotap_timestamp: RadiotapTimestamp,
  uapsd_queues: u8,
  uapsd_max_sp_len: u8,
  max_mtu: u32,
  tx_sk_pacing_shift: u32,               // default 7
  max_listen_interval: u32,
  max_nan_de_entries: u32,
  weight_multiplier: u32,
  max_signal: i8,
  netdev_features: NetdevFeatures,
}

trait Ieee80211Ops {                     // driver vtable
  fn tx(...);
  fn start(...);
  fn stop(...);
  fn config(...);
  fn add_interface(...);
  fn remove_interface(...);
  fn configure_filter(...);
  fn wake_tx_queue(...);
  // Optional (validated combinations):
  fn sta_state(...);                     // XOR sta_add/sta_remove
  fn sta_add(...);
  fn sta_remove(...);
  fn link_info_changed(...);             // iff vif_cfg_changed
  fn vif_cfg_changed(...);
  fn bss_info_changed(...);              // mutex with link_info_changed
  fn add_chanctx(...);                   // all-or-none real chanctx
  fn remove_chanctx(...);
  fn change_chanctx(...);
  fn assign_vif_chanctx(...);
  fn unassign_vif_chanctx(...);
  fn set_key(...);                       // absent ⇒ SW-crypto
  fn hw_scan(...);
  fn remain_on_channel(...);
  fn suspend(...);                       // required if wowlan
  fn resume(...);
  fn tdls_channel_switch(...);
  fn tdls_cancel_channel_switch(...);
  fn tdls_recv_channel_switch(...);
  fn set_frag_threshold(...);            // required if SUPPORTS_TX_FRAG
  fn start_nan(...);                     // required if NAN iface
  fn stop_nan(...);
  fn add_nan_func(...);
  fn del_nan_func(...);
  fn ipv6_addr_change(...);
  // ... full vtable ~120 entries
}
```

`Mac80211::alloc_hw_nm(priv_data_len, ops, requested_name) -> Option<&Ieee80211Hw>`:
1. /* REQ-3 op-mask validation */ — return None on any failure.
2. priv_size = align(sizeof(Mac80211Local), NETDEV_ALIGN) + priv_data_len.
3. wiphy = Wiphy::new_nm(&mac80211_config_ops, priv_size, requested_name)?.
4. /* wiphy defaults: mgmt_stypes, flags, features, ext_features per REQ-4 */
5. local = wiphy.priv_mut::<Mac80211Local>().
6. StaInfo::init(local).map_err(|| wiphy.free()).ok()?.
7. local.hw.wiphy = wiphy; local.hw.priv = (local as *mut u8).add(align(sizeof(Local), NETDEV_ALIGN)).
8. local.ops = ops; local.emulate_chanctx = emulate_chanctx.
9. /* Defaults per REQ-4 */
10. /* Lists / locks / workitems / tasklets / skb_queues init */
11. /* RoC + LED names */
12. Some(&local.hw).

`Mac80211::register_hw(hw) -> Result<()>`:
1. /* REQ-5 cap consistency checks */
2. /* Per-band scan: default chandef, supp_* flags, scan_ies_len computation */
3. /* HT/VHT/HE require ≥ IEEE80211_NUM_ACS queues; EHT ⇒ HE; UHR ⇒ EHT */
4. /* AP_VLAN, MONITOR mode addition */
5. /* int_scan_req alloc */
6. /* Mesh, control-port-protocol, signal-type, ext features, scan_ies_len-adjust */
7. Mac80211::init_cipher_suites(local)?
8. /* remain_on_channel default; TDLS_EXTERNAL_SETUP; multi-BSSID; max_num_csa_counters */
9. /* queues clamp; workqueue alloc */
10. /* tx_headroom, defaults */
11. WepEngine::init(local).
12. local.hw.conf.flags = IEEE80211_CONF_IDLE.
13. Mac80211::led_init(local).
14. Mac80211::txq_setup_flows(local)?
15. /* rate-control init under RTNL */
16. /* VHT EXT NSS BW sband adjust */
17. /* ASSOC_FRAME_ENCRYPTION → EPPKE */
18. wiphy_register(local.hw.wiphy)?
19. /* debugfs, wbrf support */
20. /* Default wlan%d STA iface */
21. /* register inet/inet6 notifiers */
22. Ok(())

`Mac80211::unregister_hw(hw)`:
1. /* tasklet_kill all */
2. /* unreg notifiers */
3. /* RTNL: ieee80211_remove_interfaces + ieee80211_txq_teardown_flows */
4. /* wiphy_lock: cancel work items */
5. cancel_work_sync(restart_work).
6. ieee80211_clear_tx_pending(local).
7. RateControl::deinitialize(local).
8. /* skb_queue / skb_queue_unreliable must be empty */
9. /* purge both */
10. wiphy_unregister. destroy_workqueue. led_exit. kfree(int_scan_req).

`Mac80211::free_hw(hw)`:
1. mutex_destroy(iflist_mtx).
2. Idr::for_each(ack_status_frames, |skb| { WARN_ONCE; SkBuff::kfree(skb); }).
3. idr_destroy(ack_status_frames).
4. StaInfo::stop(local).
5. Led::free_names(local).
6. /* Free per-band sbands per sband_allocated bitmap */
7. Wiphy::free(local.hw.wiphy).

`Mac80211::tasklet_handler(t)`:
1. local = from_tasklet(local, t, tasklet).
2. Mac80211::handle_queued_frames(local).

`Mac80211::handle_queued_frames(local)`:
1. loop:
   - skb = local.skb_queue.dequeue().or_else(|| local.skb_queue_unreliable.dequeue()).
   - if None: break.
   - match skb.pkt_type:
     - IEEE80211_RX_MSG => { skb.pkt_type = 0; ieee80211_rx(&local.hw, skb); }
     - IEEE80211_TX_STATUS_MSG => { skb.pkt_type = 0; ieee80211_tx_status_skb(&local.hw, skb); }
     - _ => { WARN(1, "unknown type {}", skb.pkt_type); SkBuff::dev_kfree(skb); }

`Mac80211::configure_filter(local)`:
1. /* REQ-12 mask aggregation */
2. spin_lock_bh(&local.filter_lock).
3. changed = local.filter_flags ^ new_flags.
4. mc = local.ops.prepare_multicast(local, &local.mc_list).
5. spin_unlock_bh.
6. new_flags |= 1 << 31.
7. local.ops.configure_filter(local, changed, &mut new_flags, mc).
8. WARN_ON(new_flags & (1 << 31)).
9. local.filter_flags = new_flags & !(1u32 << 31).

`Mac80211::restart_hw(hw)`:
1. local = hw_to_local(hw).
2. trace_api_restart_hw(local).
3. wiphy_info("Hardware restart was requested").
4. Mac80211::stop_queues_by_reason(hw, IEEE80211_MAX_QUEUE_MAP, QUEUE_STOP_REASON_SUSPEND, false).
5. local.in_reconfig = true; compiler_fence(SeqCst).
6. queue_work(system_freezable_wq, &local.restart_work).

`Mac80211::init_cipher_suites(local) -> Result<()>`:
1. have_mfp = ieee80211_hw_check(&local.hw, MFP_CAPABLE).
2. if ieee80211_hw_check(&local.hw, SW_CRYPTO_CONTROL) ∧ fips_enabled: return Err(EINVAL).
3. if SW_CRYPTO_CONTROL ∧ wiphy.cipher_suites == null: return Err(EINVAL).
4. if fips_enabled ∨ wiphy.cipher_suites == null:
   - wiphy.cipher_suites = &DEFAULT_SUITES.
   - n = ARRAY_LEN(DEFAULT_SUITES).
   - if !have_mfp: n -= 4.
   - if fips_enabled: wiphy.cipher_suites = &DEFAULT_SUITES[3..]; n -= 3.
   - wiphy.n_cipher_suites = n.
5. Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `alloc_hw_nm_rejects_null_mandatory_ops` | INVARIANT | per-REQ-3: missing mandatory op → None. |
| `alloc_hw_nm_priv_alignment` | INVARIANT | per-REQ-4: hw.priv = local + ALIGN(sizeof(local), NETDEV_ALIGN). |
| `register_hw_ht_requires_acs_queues` | INVARIANT | per-REQ-5: HT/VHT/HE ⇒ queues ≥ IEEE80211_NUM_ACS. |
| `register_hw_eht_requires_he` | INVARIANT | per-REQ-5: EHT ⇒ HE supported on same sband. |
| `register_hw_uhr_requires_eht` | INVARIANT | per-REQ-5: UHR ⇒ EHT. |
| `register_hw_mlo_disallows_emulate_chanctx` | INVARIANT | per-REQ-5: MLO ∧ emulate_chanctx ⇒ -EINVAL. |
| `unregister_hw_kills_tasklets_first` | INVARIANT | per-REQ-6: tasklet_kill before remove_interfaces. |
| `configure_filter_sentinel_cleared` | INVARIANT | per-REQ-12: bit-31 cleared post drv_configure_filter. |
| `init_cipher_suites_fips_strips_rc4` | INVARIANT | per-REQ-15: fips_enabled ⇒ default list starts past WEP/TKIP. |
| `handle_queued_frames_drains_both_queues` | INVARIANT | per-REQ-11: skb_queue drained then skb_queue_unreliable. |

### Layer 2: TLA+

`net/mac80211/main.tla`:
- States: per-hw {Unallocated, Allocated, Registered, Running, Restarting, Unregistered}.
- Actions: alloc_hw_nm, register_hw, restart_hw → restart_work → reconfig → resume, unregister_hw, free_hw, tasklet_fires, configure_filter.
- Properties:
  - `safety_no_register_before_alloc` — per-hw: register_hw requires Allocated.
  - `safety_no_double_register` — per-hw: register_hw idempotent-or-failing on Registered.
  - `safety_skb_queue_empty_at_unregister` — per-REQ-6: tasklet killed before queue purge; WARN if non-empty.
  - `safety_in_reconfig_implies_queues_stopped` — per-REQ-8: in_reconfig=true ⇒ all-AC queues stopped reason=SUSPEND.
  - `safety_tasklets_dead_at_free` — per-REQ-7: free_hw after unregister_hw.
  - `liveness_restart_work_completes` — per-restart_hw: queued restart_work eventually runs ieee80211_reconfig.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mac80211::alloc_hw_nm` post: ret.is_some() ⇒ all mandatory ops non-null ∧ priv 32-byte aligned | `Mac80211::alloc_hw_nm` |
| `Mac80211::register_hw` post: Ok ⇒ wiphy_register'd ∧ rate_ctrl initialized ∧ default STA iface present (if STATION supported ∧ !NO_AUTO_VIF) | `Mac80211::register_hw` |
| `Mac80211::unregister_hw` post: tasklets killed ∧ notifiers unregistered ∧ workqueue destroyed ∧ skb queues empty | `Mac80211::unregister_hw` |
| `Mac80211::handle_queued_frames` post: skb_queue / skb_queue_unreliable both empty | `Mac80211::handle_queued_frames` |
| `Mac80211::configure_filter` post: local.filter_flags == new_flags & !(1<<31) | `Mac80211::configure_filter` |
| `Mac80211::init_cipher_suites` post: wiphy.cipher_suites non-NULL ∧ n_cipher_suites > 0 | `Mac80211::init_cipher_suites` |
| `Mac80211::restart_hw` post: in_reconfig=true ∧ restart_work queued ∧ all queues stopped reason=SUSPEND | `Mac80211::restart_hw` |

### Layer 4: Verus/Creusot functional

`Per-driver lifecycle: alloc_hw_nm → register_hw → (operating: tx/rx/restart/filter cycles) → unregister_hw → free_hw` semantic equivalence: per `Documentation/networking/mac80211-injection.rst`, `Documentation/driver-api/80211/mac80211.rst`, and `iwlwifi` / `ath11k` / `mac80211_hwsim` in-tree reference users.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

mac80211-core reinforcement:

- **Per-op mandatory-mask validated in alloc_hw_nm** — defense against per-driver-incomplete vtable panic on first packet.
- **Per-mixed-emulate-chanctx detected at alloc_hw_nm** — defense against per-half-emulated chanctx UB.
- **Per-MLO cap mandate (HAS_RATE_CONTROL, AMPDU_AGGREGATION, MFP_CAPABLE, AP_LINK_PS, …)** — defense against per-MLO-without-mandatory-cap behavioral divergence.
- **Per-HT/VHT/HE ⇒ ≥ IEEE80211_NUM_ACS queues** — defense against per-QoS-without-queues TX corruption.
- **Per-EHT ⇒ HE ∧ UHR ⇒ EHT cap ordering** — defense against per-PHY-cap inconsistency.
- **Per-tasklet_kill before iflist teardown** — defense against per-bottom-half UAF on driver remove.
- **Per-skb_queue WARN-not-empty at unregister** — defense against per-leak-on-unregister.
- **Per-filter bit-31 sentinel** — defense against per-driver-doesn't-honor-flags-mask.
- **Per-FIPS-strips-RC4 (WEP+TKIP)** — defense against per-FIPS-noncompliance.
- **Per-SW_CRYPTO_CONTROL+FIPS rejected** — defense against per-driver-software-crypto under FIPS.
- **Per-ack_status_frames idr_destroy + WARN_ONCE** — defense against per-leaked-ack-frame.
- **Per-rate-control init under RTNL** — defense against per-concurrent-iface-mode-change race.
- **Per-default-STA iface gated on NO_AUTO_VIF** — defense against per-test-driver unwanted netdev.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/mac80211/rx.c RX handler chain internals (covered in `rx.md` Tier-3)
- net/mac80211/tx.c TX handler chain + airtime fairness (covered in `tx.md` Tier-3)
- net/mac80211/mlme.c station-MLME (covered in `mlme.md` Tier-3)
- net/mac80211/scan.c scanning (covered in `scan.md` Tier-3)
- net/mac80211/sta_info.c per-peer state (covered in `sta_info.md` Tier-3)
- net/mac80211/key.c key management (covered in `key.md` Tier-3)
- net/mac80211/mesh*.c mesh-MLME / HWMP / PLINK (covered in `mesh.md` Tier-3)
- net/mac80211/agg-*.c BA / A-MPDU aggregation (covered in `agg.md` Tier-3)
- net/mac80211/wep.c, tkip.c, ccmp.c, gcmp.c, aes_*.c crypto (covered in `crypto.md` Tier-3)
- net/mac80211/chan.c channel context (covered in `chan.md` Tier-3)
- net/mac80211/rate.c rate-control glue + minstrel (covered in `rate.md` Tier-3)
- net/mac80211/util.c queue stop / reconfig (covered in `util.md` Tier-3)
- net/wireless/* cfg80211 (covered in `cfg80211.md` Tier-3)
- Implementation code
