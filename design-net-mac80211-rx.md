---
title: "Tier-3: net/mac80211/rx.c — mac80211 RX path"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The mac80211 RX path consumes an `sk_buff` delivered by a low-level driver via `ieee80211_rx_napi()` / `ieee80211_rx_irqsafe()` / `ieee80211_rx_list()` and walks it through a fixed chain of `ieee80211_rx_h_*` handlers that progressively decapsulate, defragment, decrypt, reorder (per A-MPDU TID), and deliver the frame either up to `netif_receive_skb_list` (data) or to the appropriate mgmt path (action/mgmt/control). Per-`struct ieee80211_rx_data` accumulates the working state (sdata, sta, link, key, security_idx, seqno_idx). Per-`struct ieee80211_rx_status` (in `skb->cb`) carries driver-provided per-MPDU metadata: band, freq, signal, encoding (HT/VHT/HE/EHT/UHR/Legacy), rate_idx/nss, flags (`RX_FLAG_DECRYPTED`, `_IV_STRIPPED`, `_PN_VALIDATED`, `_MIC_STRIPPED`, `_AMSDU_MORE`, `_DUP_VALIDATED`, `_FAILED_PLCP_CRC`, `_8023`). Per-A-MPDU reorder buffer (`struct tid_ampdu_rx`) holds out-of-order MPDUs in a ring of `buf_size` slots keyed by sequence-number modulo. Per-fast-rx opt-in: when `IEEE80211_RX_AMSDU` and `RX_FLAG_DUP_VALIDATED|PN_VALIDATED|DECRYPTED` constraints satisfied, `ieee80211_invoke_fast_rx()` bypasses the slow chain. Per-FCS check: driver responsibility; mac80211 honors `RX_FLAG_FAILED_PLCP_CRC` by dropping. Critical for: replay-protection (PN), authorization gating, A-MPDU reordering correctness, deauth-on-MIC-fail, MFP/SA-Query.

This Tier-3 covers `net/mac80211/rx.c` (~5688 lines).

### Acceptance Criteria

- [ ] AC-1: ieee80211_rx_napi called outside softirq: WARN_ON_ONCE.
- [ ] AC-2: status->band ≥ NUM_NL80211_BANDS: drop with WARN_ON.
- [ ] AC-3: HT rate with rate_idx > 76: WARN + drop in ieee80211_rx_list.
- [ ] AC-4: QoS-data unicast: h_check_dup detects retry+same seq_ctrl → RX_DROP_U_DUP, increments `dot11FrameDuplicateCount`.
- [ ] AC-5: Class-3 data from !ASSOC STA in AP mode: h_check → RX_DROP_U_SPURIOUS, optional cfg80211 notify.
- [ ] AC-6: A-MPDU MPDU within window: ieee80211_sta_manage_reorder_buf inserts at index = seq mod buf_size, increments stored_mpdu_num.
- [ ] AC-7: A-MPDU MPDU beyond window (seq ≥ head + buf_size): release frames up to new head, then insert.
- [ ] AC-8: A-MPDU gap exceeds HT_RX_REORDER_BUF_TIMEOUT: reorder_timer fires → release_reorder_timeout drains skipped slots.
- [ ] AC-9: A-MPDU duplicate (same seq already stored): dev_kfree_skb, no buffer update.
- [ ] AC-10: BAR (block-ack request): h_ctrl extracts start_seq_num, releases reorder frames up to start_seq_num, resets session_timer.
- [ ] AC-11: CCMP data with RX_FLAG_DECRYPTED | _IV_STRIPPED: h_decrypt selects PTK, skips cipher kernel, returns CONTINUE.
- [ ] AC-12: Protected data without matching key: h_decrypt → RX_DROP_U_DECRYPT_FAIL / _NO_KEY_ID / _UNPROTECTED.
- [ ] AC-13: Fast-rx opt-in: RX_FLAG_DUP_VALIDATED + _PN_VALIDATED + _DECRYPTED + RFC1042 SNAP + DS-match: ieee80211_invoke_fast_rx delivers via ieee80211_rx_8023.
- [ ] AC-14: ieee80211_drop_unencrypted on plaintext data when key exists: drop.
- [ ] AC-15: TKIP MIC failure: h_michael_mic_verify triggers cfg80211 MIC-failure event; countermeasures path.

### Architecture

```
struct RxData {
  list: Option<*ListHead>,
  skb: *Skb,
  local: *Local,
  sdata: *SubIfData,
  link: *LinkData,
  sta: Option<*StaInfo>,
  link_sta: Option<*LinkStaInfo>,
  key: Option<*Key>,
  flags: u32,                              // IEEE80211_RX_*
  seqno_idx: i32,                          // 0..15 TID, 16 non-QoS
  security_idx: i32,                       // 0..15 TID, 16 CCMP-mgmt
  link_id: i32,                            // -1 = unspec
}

struct TidAmpduRx {
  reorder_lock: SpinLock,
  reorder_buf_filtered: u64,               // bitmap of filtered slots
  reorder_buf: *[SkbQueue],                // length = buf_size
  reorder_time: *[u64],                    // jiffies per slot
  sta: *StaInfo,
  session_timer: Timer,
  reorder_timer: Timer,
  last_rx: u64,
  head_seq_num: u16,                       // next-expected SN (12-bit field zero-extended)
  stored_mpdu_num: u16,
  ssn: u16,
  buf_size: u16,
  timeout: u16,                            // TUs
  tid: u8,
  flags: u8,                               // auto_seq | removed | started
}

struct RxStatus {                          // shared layout with kernel ieee80211_rx_status
  mactime: u64,
  device_timestamp: u32,
  boottime_ns: u64,
  freq: u16,
  freq_offset: u16,
  band: NL80211Band,
  signal: i8,
  chain_signal: [i8; IEEE80211_MAX_CHAINS],
  encoding: MacRxEncoding,                  // LEGACY | HT | VHT | HE | EHT | UHR
  rate_idx: u8,
  nss: u8,
  bw: RateInfoBw,
  flag: u32,                                // RX_FLAG_*
  rx_flags: u8,                             // IEEE80211_RX_*
  link_id: u8,
  link_valid: bool,
  // encoding-specific unions (he, eht, uhr) ...
}
```

`Rx::napi(hw, pubsta, skb, napi)`:
1. rcu_read_lock.
2. Rx::list(hw, pubsta, skb, &list).
3. rcu_read_unlock.
4. if napi.is_some():
   - for each (skb, tmp) in list: skb_list_del_init(skb); napi_gro_receive(napi, skb).
5. else: netif_receive_skb_list(&list).

`Rx::list(hw, pubsta, skb, list)`:
1. WARN_ON_ONCE(softirq_count() == 0).
2. status = IEEE80211_SKB_RXCB(skb).
3. if status.band >= NUM_NL80211_BANDS: drop.
4. sband = hw.wiphy.bands[status.band].
5. if quiescing | suspended | in_reconfig | !started: drop.
6. /* Validate rate per encoding (see REQ-7) */
7. if status.link_id >= IEEE80211_LINK_UNSPECIFIED: drop.
8. status.rx_flags = 0.
9. kcov_remote_start.
10. if !RX_FLAG_8023: skb = ieee80211_rx_monitor(local, skb, rate).
11. if skb:
    - if RX_FLAG_8023: Rx::handle_8023(hw, pubsta, skb, list).
    - else: Rx::handle_packet(hw, pubsta, skb, list).
12. kcov_remote_stop.

`Rx::invoke_handlers(rx)`:
1. reorder_release = SkbQueue::new().
2. CALL_RXH(h_check_dup).
3. CALL_RXH(h_check).
4. Rx::reorder_ampdu(rx, &reorder_release).
5. Rx::run_handlers(rx, &reorder_release).
6. /* CALL_RXH on DROP/QUEUED: ieee80211_rx_handlers_result */

`Rx::run_handlers(rx, frames)`:
1. spin_lock_bh(&rx.local.rx_path_lock).
2. while skb = __skb_dequeue(frames):
   - rx.skb = skb.
   - if !rx.link: res = RX_DROP_U_NO_LINK; goto rxh_next.
   - CALL_RXH(h_check_more_data).
   - CALL_RXH(h_uapsd_and_pspoll).
   - CALL_RXH(h_sta_process).
   - CALL_RXH(h_decrypt).
   - CALL_RXH(h_defragment).
   - CALL_RXH(h_michael_mic_verify).
   - CALL_RXH(h_amsdu).
   - CALL_RXH(h_data).
   - res = Rx::h_ctrl(rx, frames); if != CONTINUE: goto rxh_next.
   - CALL_RXH(h_mgmt_check).
   - CALL_RXH(h_action).
   - CALL_RXH(h_userspace_mgmt).
   - CALL_RXH(h_action_post_userspace).
   - CALL_RXH(h_action_return).
   - CALL_RXH(h_ext).
   - CALL_RXH(h_mgmt).
   - rxh_next: Rx::handlers_result(rx, res).
3. spin_unlock_bh(&rx.local.rx_path_lock).

`Rx::manage_reorder_buf(sdata, tid_agg_rx, skb, frames) -> bool`:
1. spin_lock(&tid_agg_rx.reorder_lock).
2. if tid_agg_rx.auto_seq:
   - tid_agg_rx.auto_seq = false; tid_agg_rx.ssn = mpdu_seq_num; tid_agg_rx.head_seq_num = mpdu_seq_num.
3. buf_size = tid_agg_rx.buf_size; head_seq_num = tid_agg_rx.head_seq_num.
4. if !tid_agg_rx.started:
   - if ieee80211_sn_less(mpdu_seq_num, head_seq_num): ret = false; out.
   - tid_agg_rx.started = true.
5. if ieee80211_sn_less(mpdu_seq_num, head_seq_num): dev_kfree_skb(skb); out.
6. if !ieee80211_sn_less(mpdu_seq_num, head_seq_num + buf_size):
   - head_seq_num = sn_inc(sn_sub(mpdu_seq_num, buf_size)).
   - Rx::release_reorder_frames(sdata, tid_agg_rx, head_seq_num, frames).
7. index = mpdu_seq_num mod tid_agg_rx.buf_size.
8. if Rx::reorder_ready(tid_agg_rx, index): dev_kfree_skb(skb); out.
9. if mpdu_seq_num == tid_agg_rx.head_seq_num ∧ tid_agg_rx.stored_mpdu_num == 0:
   - if !RX_FLAG_AMSDU_MORE: tid_agg_rx.head_seq_num = sn_inc(head_seq_num).
   - ret = false; out.
10. __skb_queue_tail(&tid_agg_rx.reorder_buf[index], skb).
11. if !RX_FLAG_AMSDU_MORE:
    - tid_agg_rx.reorder_time[index] = jiffies.
    - tid_agg_rx.stored_mpdu_num++.
    - Rx::sta_reorder_release(sdata, tid_agg_rx, frames).
12. out: spin_unlock; return ret.

`Rx::sta_reorder_release(sdata, tid_agg_rx, frames)`:
1. index = head_seq_num mod buf_size.
2. if !reorder_ready(tid_agg_rx, index) ∧ stored_mpdu_num:
   - /* head missing; scan for ready slot beyond gap */
   - skipped = 1.
   - for j = (index+1) mod buf_size; j != index; j = (j+1) mod buf_size:
     - if !reorder_ready(tid_agg_rx, j): skipped++; continue.
     - if skipped ∧ !time_after(jiffies, reorder_time[j] + HT_RX_REORDER_BUF_TIMEOUT): goto set_release_timer.
     - /* Reap intermediate incomplete A-MSDUs */
     - release_reorder_frame(sdata, tid_agg_rx, j, frames).
     - tid_agg_rx.head_seq_num = (head_seq_num + skipped) & IEEE80211_SN_MASK.
     - skipped = 0.
3. else: while reorder_ready(tid_agg_rx, index):
   - release_reorder_frame(sdata, tid_agg_rx, index, frames).
   - index = head_seq_num mod buf_size.
4. if stored_mpdu_num:
   - set_release_timer: if !removed: mod_timer(reorder_timer, reorder_time[j] + 1 + HT_RX_REORDER_BUF_TIMEOUT).
5. else: timer_delete(reorder_timer).

`Rx::h_decrypt(rx) -> Result`:
1. if ieee80211_is_ext(fc): return CONTINUE.
2. rx.key = None.
3. if rx.sta:
   - sta_ptk = rcu_deref(rx.sta.ptk[rx.sta.ptk_idx]).
   - if has_protected(fc) ∧ !RX_FLAG_IV_STRIPPED: ptk_idx = rcu_deref(rx.sta.ptk[ieee80211_get_keyid(skb)]).
4. if !has_protected(fc): mmie_keyidx = ieee80211_get_mmie_keyidx(skb).
5. /* Selector cascade: unicast→PTK; mmie+beacon→BIGTK; mmie+mgmt→IGTK; !protected→default; multicast→link/sdata default */
6. if rx.key:
   - if KEY_FLAG_TAINTED: return RX_DROP_U_KEY_TAINTED.
7. else: return RX_DROP_U_UNPROTECTED.
8. /* Cipher dispatch: */
9. match rx.key.conf.cipher:
   - WEP40/104 → crypto_wep_decrypt(rx).
   - TKIP → crypto_tkip_decrypt(rx).
   - CCMP / CCMP_256 → crypto_ccmp_decrypt(rx, MIC_LEN).
   - AES_CMAC / BIP_CMAC_256 → crypto_aes_cmac_decrypt(rx, MIC_LEN).
   - BIP_GMAC_128/256 → crypto_aes_gmac_decrypt(rx).
   - GCMP / GCMP_256 → crypto_gcmp_decrypt(rx).
   - _ → RX_DROP_U_BAD_CIPHER.
10. status.flag |= RX_FLAG_DECRYPTED.
11. if beacon ∧ RX_RES_IS_UNUSABLE(result): cfg80211_rx_unprot_mlme_mgmt.
12. return result.

`Rx::invoke_fast_rx(rx, fast_rx) -> bool`:
1. if !RX_FLAG_DUP_VALIDATED: return false.
2. FAST_RX_CRYPT_FLAGS = PN_VALIDATED | DECRYPTED.
3. if fast_rx.key ∧ (status.flag & CRYPT_FLAGS) != CRYPT_FLAGS: return false.
4. if !is_data_present(fc): return false.
5. if is_frag(hdr): return false.
6. if vif_addr != hdr.addr1: return false.
7. if (fc & (FROMDS|TODS)) != fast_rx.expected_ds_bits: return false.
8. if fast_rx.key ∧ !RX_FLAG_IV_STRIPPED: snap_offs += CCMP_HDR_LEN.
9. if !mesh ∧ !AMSDU: pull SNAP; verify rfc1042 == fast_rx.rfc1042_hdr; reject TDLS/EAPOL.
10. /* commit point */
11. if rx.key ∧ !RX_FLAG_MIC_STRIPPED: pskb_trim(icv_len) — drop on fail.
12. if rx.key ∧ !has_protected(fc): drop.
13. if AMSDU: __ieee80211_rx_h_amsdu(rx, snap_offs - hdrlen).
14. else: ether_addr_copy(da, sa); skb header convert; ieee80211_rx_mesh_data; ieee80211_rx_8023(rx, fast_rx, orig_len).
15. return true.

### Out of Scope

- TKIP / CCMP / GCMP / GMAC / WEP RX cipher kernels (covered in `net/mac80211/wpa.md` and `net/mac80211/aead_api.md`).
- A-MPDU BA session setup (addba/delba) (covered in `net/mac80211/agg-rx.md`).
- Mesh-specific RX handling (`ieee80211_rx_mesh_check`, `ieee80211_rx_mesh_data`) (covered in `net/mac80211/mesh.md`).
- MLME mgmt frame handling beyond rx_h_mgmt entry (covered in `net/mac80211/mlme.md`).
- Scan beacon parsing pipeline (covered in `net/mac80211/scan.md`).
- Monitor / radiotap export (`ieee80211_rx_monitor`) details (covered in `net/mac80211/radiotap.md` if expanded).
- cfg80211 RX delivery (`cfg80211_rx_mgmt`, `cfg80211_report_obss_beacon_khz`).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ieee80211_rx_data` | per-frame rx working state | `RxData` |
| `struct ieee80211_rx_status` | per-skb rx control block | `RxStatus` (shared layout) |
| `struct tid_ampdu_rx` | per-(sta,tid) reorder buffer | `TidAmpduRx` |
| `struct ieee80211_fast_rx` | per-sta cached fast-rx params | `FastRx` |
| `ieee80211_rx_napi()` | driver entry (NAPI) | `Rx::napi` |
| `ieee80211_rx_irqsafe()` | driver entry (hardirq) | `Rx::irqsafe` |
| `ieee80211_rx_list()` | per-frame initial validation | `Rx::list` |
| `__ieee80211_rx_handle_packet()` | per-frame 802.11 dispatch | `Rx::handle_packet` |
| `__ieee80211_rx_handle_8023()` | per-frame HW-decap 802.3 dispatch | `Rx::handle_8023` |
| `ieee80211_prepare_and_rx_handle()` | per-(skb,sdata) prepare + run | `Rx::prepare_and_handle` |
| `ieee80211_invoke_rx_handlers()` | per-frame: dup, check, reorder, handlers | `Rx::invoke_handlers` |
| `ieee80211_rx_handlers()` | per-mpdu post-reorder chain | `Rx::run_handlers` |
| `ieee80211_rx_h_check_dup()` | per-frame: 802.11 retry dedupe | `Rx::h_check_dup` |
| `ieee80211_rx_h_check()` | per-frame: class-3 / association gating | `Rx::h_check` |
| `ieee80211_rx_h_check_more_data()` | per-frame: PS-poll FSM | `Rx::h_check_more_data` |
| `ieee80211_rx_h_uapsd_and_pspoll()` | per-frame: AP-side uAPSD trigger | `Rx::h_uapsd_and_pspoll` |
| `ieee80211_rx_h_sta_process()` | per-frame: STA stats + ps transitions | `Rx::h_sta_process` |
| `ieee80211_rx_h_decrypt()` | per-frame: cipher-suite dispatch | `Rx::h_decrypt` |
| `ieee80211_rx_h_defragment()` | per-frame: MPDU defragmentation | `Rx::h_defragment` |
| `ieee80211_rx_h_michael_mic_verify()` (wpa.c) | per-frame: TKIP MIC check | `Rx::h_michael_mic_verify` |
| `ieee80211_rx_h_amsdu()` | per-frame: A-MSDU subframe split | `Rx::h_amsdu` |
| `ieee80211_rx_h_data()` | per-frame: 802.11 → 802.3 + deliver | `Rx::h_data` |
| `ieee80211_rx_h_ctrl()` | per-frame: BAR processing | `Rx::h_ctrl` |
| `ieee80211_rx_h_mgmt_check()` | per-frame: mgmt minimum / beacon report | `Rx::h_mgmt_check` |
| `ieee80211_rx_h_action()` | per-frame: action dispatch | `Rx::h_action` |
| `ieee80211_rx_h_userspace_mgmt()` | per-frame: cfg80211 mgmt-rx | `Rx::h_userspace_mgmt` |
| `ieee80211_rx_h_action_post_userspace()` | per-frame: post-userspace actions | `Rx::h_action_post_userspace` |
| `ieee80211_rx_h_action_return()` | per-frame: send action response | `Rx::h_action_return` |
| `ieee80211_rx_h_ext()` | per-frame: 802.11 extension frames | `Rx::h_ext` |
| `ieee80211_rx_h_mgmt()` | per-frame: deauth/assoc/probe/etc. | `Rx::h_mgmt` |
| `ieee80211_rx_reorder_ampdu()` | per-frame: A-MPDU reorder gate | `Rx::reorder_ampdu` |
| `ieee80211_sta_manage_reorder_buf()` | per-MPDU: store / release reorder buf | `Rx::manage_reorder_buf` |
| `ieee80211_sta_reorder_release()` | per-(sta,tid): drain ring on timer/release | `Rx::sta_reorder_release` |
| `ieee80211_release_reorder_frame()` | per-slot: drain single MPDU/A-MSDU set | `Rx::release_reorder_frame` |
| `ieee80211_release_reorder_frames()` | per-window-advance: bulk drain | `Rx::release_reorder_frames` |
| `ieee80211_release_reorder_timeout()` | per-timer: BAR-timeout drain | `Rx::release_reorder_timeout` |
| `ieee80211_rx_reorder_ready()` | per-slot: ready predicate | `Rx::reorder_ready` |
| `ieee80211_accept_frame()` | per-frame: vif address-match filter | `Rx::accept_frame` |
| `ieee80211_invoke_fast_rx()` | per-frame: fast-rx opt-in dispatch | `Rx::invoke_fast_rx` |
| `ieee80211_rx_for_interface()` | per-(sdata,skb) iter helper | `Rx::for_interface` |
| `ieee80211_rx_data_set_sta()` / `_set_link()` | per-frame: sta/link resolution | `Rx::set_sta` / `set_link` |
| `ieee80211_parse_qos()` | per-frame: TID + seqno_idx assignment | `Rx::parse_qos` |
| `ieee80211_verify_alignment()` | per-frame: alignment check | `Rx::verify_alignment` |
| `ieee80211_drop_unencrypted()` / `_mgmt()` | per-frame: drop-unencrypted policy | `Rx::drop_unencrypted` / `_mgmt` |
| `ieee80211_802_1x_port_control()` | per-frame: 802.1X port gate | `Rx::port_control` |

### compatibility contract

REQ-1: struct ieee80211_rx_data:
- list: optional per-batch list for netif_receive_skb_list.
- skb: per-frame primary skb.
- local: per-`struct ieee80211_local` back-pointer.
- sdata: per-`struct ieee80211_sub_if_data` for receiving vif.
- link: per-`struct ieee80211_link_data` (MLD link, may be sdata->deflink).
- sta: optional per-`struct sta_info`.
- link_sta: optional per-`struct link_sta_info`.
- key: optional per-`struct ieee80211_key` selected by `h_decrypt`.
- flags: per-`IEEE80211_RX_BEACON_REPORTED`.
- seqno_idx: 0..15 (TID) or 16 (non-QoS) — index into `last_seq_ctrl[]`.
- security_idx: 0..15 (TID) or 16 (CCMP-mgmt) — index into RX PN arrays.
- link_id: 0..14 or -1 (unspec).

REQ-2: struct ieee80211_rx_status (IEEE80211_SKB_RXCB):
- mactime / device_timestamp.
- boottime_ns.
- freq / freq_offset.
- band: per-`enum nl80211_band`.
- signal: dBm (when RX_FLAG_SIGNAL_DBM set).
- chain_signal[].
- rate_idx (legacy MCS / VHT idx / HE idx / EHT idx / UHR idx).
- nss.
- bw: per-`enum rate_info_bw`.
- encoding: per-`enum mac80211_rx_encoding` (LEGACY/HT/VHT/HE/EHT/UHR).
- enc_flags / he / eht / uhr unions.
- flag: per-`RX_FLAG_*` (DECRYPTED, IV_STRIPPED, MIC_STRIPPED, PN_VALIDATED, AMSDU_MORE, DUP_VALIDATED, FAILED_PLCP_CRC, NO_PSDU, 8023, ...).
- rx_flags: per-`IEEE80211_RX_*` (AMSDU, DEFERRED_RELEASE, MALFORMED_ACTION_FRM).
- link_valid / link_id.

REQ-3: struct tid_ampdu_rx (per-(sta, TID)):
- reorder_buf: per-slot `sk_buff_head` array (length `buf_size`).
- reorder_buf_filtered: bitmap of slots holding driver-filtered frames.
- reorder_time: per-slot jiffies-of-insert.
- session_timer: per-BA-session inactivity timer.
- reorder_timer: per-slot timeout timer (HT_RX_REORDER_BUF_TIMEOUT = HZ/10).
- head_seq_num: ring head (next-expected SN).
- stored_mpdu_num: total stored.
- ssn: starting sequence number negotiated.
- buf_size: ring size (≤ 1024 for HE).
- timeout: BA timeout in TUs.
- tid, auto_seq, removed, started bits.
- reorder_lock: serializes ring access.

REQ-4: RX handler chain (pre-reorder):
- /* ieee80211_invoke_rx_handlers */
- h_check_dup — per-(sta, seqno_idx) retransmission dedupe; multicast MLD STATION uses `mcast_seq_last`.
- h_check — class-3 / association gating; mesh delegates to `ieee80211_rx_mesh_check`; allow port-control-protocol carve-out.
- ieee80211_rx_reorder_ampdu — gate into per-TID reorder buffer (returns by populating `reorder_release` head).

REQ-5: RX handler chain (post-reorder, ieee80211_rx_handlers):
- For each skb dequeued from reorder_release:
- h_check_more_data — PS-poll state machine: if pspolling + FromDS + data + !MoreData ⇒ clear pspolling; else send another PSPOLL.
- h_uapsd_and_pspoll — AP-side uAPSD trigger detection.
- h_sta_process — update `link_sta->rx_stats`; sta PS transitions (FromDS data with PM bit).
- h_decrypt — pick key (PTK / link-GTK / IGTK / BIGTK via mmie_keyidx); dispatch per `key->conf.cipher`; sets `RX_FLAG_DECRYPTED`.
- h_defragment — reassemble per (addr2, seq) entries in `sdata->frags` cache; `requires_sequential_pn` enforces PN monotonicity for CCMP/GCMP fragments.
- h_michael_mic_verify — TKIP-MIC (defined in wpa.c); MIC failure triggers countermeasures via `mac80211_ev_michael_mic_failure`.
- h_amsdu — split A-MSDU into ethernet subframes via `__ieee80211_rx_h_amsdu`.
- h_data — non-A-MSDU data: `__ieee80211_data_to_8023` → mesh check → port control / 1x → `ieee80211_deliver_skb` → `RX_QUEUED`.
- /* Special: */ h_ctrl(rx, frames) — BAR processing; releases reorder frames up to `start_seq_num`.
- h_mgmt_check — minimum length, beacon broadcast-only, sw BSS-color collision detection, OBSS beacon report.
- h_action — dispatch action category (BlockAck, FT, HT, SA-Query, SPECTRUM_MGMT, MESH, TWT, etc.).
- h_userspace_mgmt — `cfg80211_rx_mgmt` for userspace consumers.
- h_action_post_userspace — actions to also run after userspace see-them.
- h_action_return — send canned action responses (SA-Query response, etc.).
- h_ext — 802.11 extension frames.
- h_mgmt — final mgmt dispatch: probe-resp/beacon to scan/MLME, deauth/disassoc/assoc/auth to MLME.

REQ-6: ieee80211_rx_napi(hw, pubsta, skb, napi):
- rcu_read_lock; ieee80211_rx_list(hw, pubsta, skb, &list); rcu_read_unlock.
- if napi: napi_gro_receive per skb; else netif_receive_skb_list(&list).

REQ-7: ieee80211_rx_list(hw, pubsta, skb, list):
- WARN_ON_ONCE(softirq_count() == 0).
- WARN_ON(status->band >= NUM_NL80211_BANDS): drop.
- /* Drop if quiescing | suspended | in_reconfig | !started */
- /* Rate validation per encoding: */
  - RX_ENC_HT: rate_idx ≤ 76.
  - RX_ENC_VHT: rate_idx ≤ 11, nss in 1..8.
  - RX_ENC_HE: rate_idx ≤ 11, nss in 1..8.
  - RX_ENC_EHT: rate_idx ≤ 15, nss in 1..8, eht.gi ≤ NL80211_RATE_INFO_EHT_GI_3_2.
  - RX_ENC_UHR: rate_idx ∈ {0..15, 17, 19, 20, 23}, nss in 1..8, uhr.gi ≤ EHT_GI_3_2; extra ELR/IM constraints.
  - RX_ENC_LEGACY: rate_idx < sband->n_bitrates.
- WARN_ON_ONCE(link_id ≥ IEEE80211_LINK_UNSPECIFIED): drop.
- status->rx_flags = 0.
- kcov_remote_start.
- if !RX_FLAG_8023: skb = ieee80211_rx_monitor(local, skb, rate) (radiotap export to monitor ifs).
- if skb:
  - data/802.3 → tpt_led_trig_rx.
  - RX_FLAG_8023 → __ieee80211_rx_handle_8023.
  - else → __ieee80211_rx_handle_packet.
- kcov_remote_stop.

REQ-8: __ieee80211_rx_handle_packet:
- Validate header length (mgmt: skb_linearize; data: pskb_may_pull hdrlen).
- ieee80211_parse_qos: sets rx.seqno_idx + security_idx from TID.
- ieee80211_verify_alignment.
- If probe-resp / beacon / s1g-beacon: ieee80211_scan_rx(local, skb).
- Data:
  - pubsta → rx_data_set_sta(rx, sta, link_id); handle MLO addr-translation via link_sta_info_get_bss(addr2); prepare_and_rx_handle(rx, skb, true).
  - else: for_each_sta_info(local, hdr->addr2, ...) — multi-vif consume / preview pattern.
- Mgmt/control: list_for_each interfaces ∧ matching vif type ≠ MONITOR/AP_VLAN → ieee80211_rx_for_interface(rx, skb, consume=last).

REQ-9: ieee80211_prepare_and_rx_handle(rx, skb, consume):
- /* Fast-rx attempt */
- if consume ∧ rx.sta:
  - fast_rx = rcu_dereference(rx.sta->fast_rx).
  - if fast_rx ∧ ieee80211_invoke_fast_rx(rx, fast_rx) ⇒ return true.
- if !ieee80211_accept_frame(rx): return false.
- if !consume: rx.skb = skb_copy(skb, GFP_ATOMIC) (preview-only).
- /* Run full handler chain */
- ieee80211_invoke_rx_handlers(rx).
- return true if consumed.

REQ-10: A-MPDU reorder (ieee80211_sta_manage_reorder_buf):
- spin_lock(reorder_lock).
- If tid_agg_rx.auto_seq (offloaded BA without known SSN): adopt mpdu_seq_num as head.
- If mpdu_seq_num < head_seq_num (signed-mod compare): out-of-window — drop unless !started (then accept and set started).
- If mpdu_seq_num ≥ head_seq_num + buf_size: advance head; release frames up to new head.
- index = mpdu_seq_num mod buf_size.
- If already stored: drop dup.
- If mpdu_seq_num == head_seq_num ∧ stored_mpdu_num == 0:
  - bypass buffer; advance head (unless RX_FLAG_AMSDU_MORE); return false (process immediately).
- Else: __skb_queue_tail(reorder_buf[index], skb); reorder_time[index] = jiffies; stored_mpdu_num++; sta_reorder_release.

REQ-11: ieee80211_sta_reorder_release:
- Drain consecutive ready slots from head.
- If gap exists: check timeout (HT_RX_REORDER_BUF_TIMEOUT = HZ/10).
  - If timeout exceeded: skip dead slot, advance head by `skipped`.
  - Else: arm reorder_timer.
- If frames remain after drain: mod_timer(reorder_timer, reorder_time[next] + 1 + HT_RX_REORDER_BUF_TIMEOUT).
- Else: timer_delete(reorder_timer).

REQ-12: ieee80211_invoke_fast_rx (fast-path opt-in):
- Require RX_FLAG_DUP_VALIDATED.
- If fast_rx.key set: require RX_FLAG_PN_VALIDATED | RX_FLAG_DECRYPTED.
- Reject !data-present / fragmented / mc / unexpected FromDS|ToDS / non-RFC1042 SNAP.
- Reject TDLS / EAPOL control-port (punt to slow path).
- Strip IV/MIC; trim ICV; do header conversion (RFC1042 → ethernet); call ieee80211_rx_8023 → netif/napi.

REQ-13: ieee80211_accept_frame(rx):
- vif address-match: hdr.addr1 vs vif.addr / multicast / link-broadcast.
- AP_VLAN station match.
- Mesh / OCB special cases.
- Returns true iff this vif is a destination.

REQ-14: FCS / PLCP check:
- Driver responsibility — mac80211 honors `RX_FLAG_FAILED_PLCP_CRC` (suppresses rate validation; uses radiotap export only).
- `RX_FLAG_NO_PSDU` + `IEEE80211_RADIOTAP_ZERO_LEN_PSDU_NOT_CAPTURED` likewise.
- Frames < 16 bytes are dropped by `ieee80211_rx_monitor`.

REQ-15: ieee80211_release_reorder_timeout(sta, tid):
- timer callback (rcu_read_lock).
- spin_lock(reorder_lock); ieee80211_sta_reorder_release(sdata, tid_agg_rx, &frames); unlock.
- If frames non-empty: drv_event_callback(BA_FRAME_TIMEOUT) + push frames into RX handler chain.

REQ-16: PN / replay protection:
- requires_sequential_pn(rx, fc) ≡ key.cipher ∈ {CCMP, CCMP_256, GCMP, GCMP_256} ∧ ieee80211_has_protected(fc).
- Defragmentation enforces PN+1 monotonicity across fragments.
- Per-TID PN counters in key.rx[security_idx].

REQ-17: Drop-unencrypted policy:
- `ieee80211_drop_unencrypted(rx, fc)`: data frame must be DECRYPTED ∨ control-port-protocol with `control_port_no_encrypt`.
- `ieee80211_drop_unencrypted_mgmt(rx)`: robust mgmt frame must be protected when STA has MFP.

REQ-18: ieee80211_802_1x_port_control(rx):
- For non-AUTHORIZED STAs, only EAPOL frames are passed up; all other data dropped.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rx_handlers_terminate` | INVARIANT | per-invoke_rx_handlers / run_handlers: chain terminates with CONTINUE | DROP | QUEUED. |
| `reorder_buf_index_in_bounds` | INVARIANT | per-manage_reorder_buf: index < tid_agg_rx.buf_size. |
| `reorder_lock_held` | INVARIANT | per-release_reorder_frame: tid_agg_rx.reorder_lock held (lockdep_assert_held). |
| `head_seq_advance_within_mask` | INVARIANT | per-manage_reorder_buf: head_seq_num stays within IEEE80211_SN_MASK (12 bits). |
| `decrypt_sets_decrypted_flag` | INVARIANT | per-h_decrypt: after dispatch, RX_FLAG_DECRYPTED set. |
| `fast_rx_requires_dup_validated` | INVARIANT | per-invoke_fast_rx: !RX_FLAG_DUP_VALIDATED ⟹ false. |
| `defragment_pn_monotonic` | INVARIANT | per-h_defragment: requires_sequential_pn ⟹ PN strictly increasing across fragments. |
| `rx_napi_softirq_only` | INVARIANT | per-rx_list: softirq_count() > 0. |
| `accept_frame_or_skipped` | INVARIANT | per-prepare_and_rx_handle: !accept_frame ⟹ no handler chain run. |

### Layer 2: TLA+

`net/mac80211/rx.tla`:
- Per-(sta, tid) reorder buffer state: ring of buf_size slots × stored_mpdu_num × head_seq_num.
- Per-RX skb lifecycle: ARRIVED → REORDERED → DECRYPTED → DEFRAGGED → DELIVERED|DROPPED.
- Per-key replay state: PN counters per (key, security_idx).
- Properties:
  - `safety_no_reorder_out_of_window` — per-MPDU: SN ≥ head ∧ SN < head + buf_size when stored.
  - `safety_no_duplicate_delivered` — per-(sta, tid): a given (seq_ctrl, retry) never delivered twice.
  - `safety_decrypt_before_data` — per-data: h_decrypt runs before h_data; protected frame dropped if decrypt fails.
  - `safety_pn_monotonic` — per-(key, security_idx): rx PN strictly increases mod 2^48.
  - `safety_class3_gating` — per-AP frame: !ASSOC STA cannot deliver data (except EAPOL via port-control).
  - `liveness_reorder_drains_on_timeout` — per-(sta, tid): gapped reorder buffer eventually flushes via reorder_timer.
  - `liveness_BAR_releases_window` — per-BAR: ieee80211_release_reorder_frames advances head to start_seq_num.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Rx::list` post: !drop ⟹ rx.skb passed to handle_packet | handle_8023 with status valid | `Rx::list` |
| `Rx::invoke_handlers` post: every skb in reorder_release passes through handler chain or is freed | `Rx::invoke_handlers` |
| `Rx::manage_reorder_buf` post: stored_mpdu_num == count(non-empty reorder_buf[]) | `Rx::manage_reorder_buf` |
| `Rx::sta_reorder_release` post: ∀ remaining slot: !ready ∨ time_after(jiffies, reorder_time[slot] + HT_RX_REORDER_BUF_TIMEOUT) is false | `Rx::sta_reorder_release` |
| `Rx::h_decrypt` post: ret CONTINUE ⟹ RX_FLAG_DECRYPTED set ∧ (rx.key.cipher matches header class) | `Rx::h_decrypt` |
| `Rx::invoke_fast_rx` post: ret true ⟹ skb consumed via ieee80211_rx_8023 | `Rx::invoke_fast_rx` |
| `Rx::h_check_dup` post: retry duplicate ⟹ RX_DROP_U_DUP ∧ counters incremented | `Rx::h_check_dup` |

### Layer 4: Verus/Creusot functional

`Per-rx skb: rx_napi → rx_list (validate rate/state) → handle_packet (parse_qos, scan_rx) → prepare_and_rx_handle (accept_frame, fast_rx | invoke_handlers) → reorder → handlers chain → deliver_skb | mgmt dispatch` semantic equivalence: per-IEEE 802.11-2020 §10 (frame reception + A-MPDU reordering) + per-IEEE 802.11i (TKIP MIC + CCMP/GCMP PN replay) + per-`Documentation/networking/mac80211.rst`.

### hardening

(Inherits row-1 features from `net/mac80211/00-overview.md` § Hardening.)

RX-path reinforcement:

- **Per-h_check_dup against retransmit replay** — defense against per-replay-storm dup-MPDU flood.
- **Per-h_check class-3 gating** — defense against per-spoofed-data from unassociated STA.
- **Per-reorder window bound (buf_size)** — defense against per-OOM via unbounded reorder ring.
- **Per-reorder_lock for all ring access** — defense against per-concurrent-driver-vs-timer reorder corruption.
- **Per-reorder timeout (HT_RX_REORDER_BUF_TIMEOUT)** — defense against per-stuck-MPDU permanent stall.
- **Per-h_decrypt KEY_FLAG_TAINTED check** — defense against per-rotated-key reuse.
- **Per-requires_sequential_pn fragments** — defense against per-CCMP/GCMP fragment-PN-shuffle replay.
- **Per-FAST_RX_CRYPT_FLAGS gate** — defense against per-fast-path crypto bypass.
- **Per-RX_FLAG_FAILED_PLCP_CRC honored** — defense against per-corrupt-PLCP rate-table OOB.
- **Per-rate validation per encoding** — defense against per-driver-bogus MCS/NSS OOB.
- **Per-rx_napi softirq check** — defense against per-cross-context skb cb corruption.
- **Per-cfg80211_rx_unprot_mlme_mgmt for unprotected beacons** — defense against per-beacon-spoof in MFP-on networks.
- **Per-ieee80211_drop_unencrypted policy** — defense against per-clear-data leak through ESS.
- **Per-Wi-Fi-Aware NAN-data unicast A2 check** — defense against per-NDP spoof.

