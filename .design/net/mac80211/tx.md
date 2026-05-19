# Tier-3: net/mac80211/tx.c — mac80211 TX path

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/mac80211/00-overview.md
upstream-paths:
  - net/mac80211/tx.c (~6545 lines)
  - net/mac80211/ieee80211_i.h (struct ieee80211_tx_data, struct txq_info)
  - net/mac80211/sta_info.h (struct tid_ampdu_tx, struct airtime_info)
  - include/net/mac80211.h (struct ieee80211_tx_info, struct ieee80211_txq, struct ieee80211_hw)
  - include/net/fq.h / include/net/fq_impl.h (FQ + CoDel queueing)
  - net/mac80211/wpa.c (ieee80211_tx_h_michael_mic_add)
-->

## Summary

The mac80211 TX path takes an `sk_buff` originating from a network device queue (or from in-stack mgmt/control originators) and walks it through a fixed chain of `ieee80211_tx_h_*` handlers that progressively annotate, fragment, encrypt, and schedule the frame for the underlying low-level driver. Per-`struct ieee80211_tx_data` accumulates the working state (sdata, sta, key, fragmented `skbs` head). Per-`struct ieee80211_tx_info` (in `skb->cb`) records driver-visible control: band, hw_queue, hw_key, rate table, tx flags (`IEEE80211_TX_CTL_*`), and the AMPDU/A-MSDU intent. Per-handler returns `TX_CONTINUE`, `TX_DROP`, or `TX_QUEUED`. Per-`ieee80211_queue_skb` deposits the frame into a per-(vif,sta,tid) `struct txq_info` whose backing store is an `fq_tin` of `fq_flow` slots (FQ-CoDel discipline). Per-`ieee80211_tx_dequeue` is the driver-facing dequeue: pulls one MPDU, finishes late handlers, applies AQL airtime accounting. Per-A-MPDU (TID-aggregated) frames take the operational fast path via `ieee80211_tx_prep_agg`. Critical for: ABI-correct EAPOL handover, deterministic encryption (CCMP/GCMP/TKIP/WEP), correct duration-NAV, and fair airtime scheduling under load.

This Tier-3 covers `net/mac80211/tx.c` (~6545 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ieee80211_tx_data` | per-frame tx working state | `TxData` |
| `struct ieee80211_tx_info` | per-skb control block (in `skb->cb`) | `TxInfo` (shared layout) |
| `struct ieee80211_txq` / `struct txq_info` | per-(vif,sta,tid) software queue | `Txq` / `TxqInfo` |
| `ieee80211_tx_prepare()` | per-frame tx_data init + agg-check | `Tx::prepare` |
| `ieee80211_tx_h_dynamic_ps()` | per-frame: wake dynamic PS / arm timer | `Tx::h_dynamic_ps` |
| `ieee80211_tx_h_check_assoc()` | per-frame: enforce STA WLAN_STA_ASSOC | `Tx::h_check_assoc` |
| `ieee80211_tx_h_ps_buf()` | per-frame: buffer to STA / BSS PS queue | `Tx::h_ps_buf` |
| `ieee80211_tx_h_multicast_ps_buf()` / `_unicast_ps_buf()` | per-mc/uc PS buffering | `Tx::h_mc_ps_buf` / `h_uc_ps_buf` |
| `ieee80211_tx_h_check_control_port_protocol()` | per-frame: EAPOL handling | `Tx::h_check_control_port_protocol` |
| `ieee80211_tx_h_select_key()` | per-frame: choose PTK/GTK/IGTK/BIGTK | `Tx::h_select_key` |
| `ieee80211_tx_h_rate_ctrl()` | per-frame: rate selection (rc80211_*) | `Tx::h_rate_ctrl` |
| `ieee80211_tx_h_sequence()` | per-frame: assign per-TID seq_ctrl | `Tx::h_sequence` |
| `ieee80211_tx_h_fragment()` | per-frame: MSDU → MPDU fragmentation | `Tx::h_fragment` |
| `ieee80211_tx_h_stats()` | per-frame: per-AC byte/packet counters | `Tx::h_stats` |
| `ieee80211_tx_h_encrypt()` | per-frame: cipher-suite dispatch | `Tx::h_encrypt` |
| `ieee80211_tx_h_calculate_duration()` | per-frame: NAV duration field | `Tx::h_calculate_duration` |
| `ieee80211_tx_h_michael_mic_add()` (wpa.c) | per-frame: TKIP MIC append | `Tx::h_michael_mic_add` |
| `ieee80211_tx_prep_agg()` | per-A-MPDU: prep / buffer to BA session | `Tx::prep_agg` |
| `invoke_tx_handlers_early()` | per-frame: pre-queue handler chain | `Tx::invoke_handlers_early` |
| `invoke_tx_handlers_late()` | per-frame: post-queue handler chain | `Tx::invoke_handlers_late` |
| `ieee80211_queue_skb()` | per-frame: enqueue → txq_info via fq | `Tx::queue_skb` |
| `ieee80211_txq_init()` | per-txq: init fq_tin + codel | `Tx::txq_init` |
| `ieee80211_txq_enqueue()` | per-frame: fq_tin_enqueue or frags tail | `Tx::txq_enqueue` |
| `ieee80211_tx_dequeue()` | driver-facing per-MPDU dequeue | `Tx::txq_dequeue` |
| `ieee80211_next_txq()` | per-AC airtime DRR scheduler | `Tx::next_txq` |
| `__ieee80211_schedule_txq()` | per-txq schedule list insert | `Tx::schedule_txq` |
| `ieee80211_txq_airtime_check()` | per-txq AQL gate (NL80211_EXT_FEATURE_AQL) | `Tx::txq_airtime_check` |
| `ieee80211_tx_frags()` | per-frame: drive hw queues, splice pending | `Tx::tx_frags` |
| `__ieee80211_tx()` | per-frame: drv_tx() dispatch | `Tx::dispatch` |
| `ieee80211_subif_start_xmit()` | netdev xmit (native 802.11) | `Tx::subif_start_xmit` |
| `ieee80211_subif_start_xmit_8023()` | netdev xmit (HW encap 802.3) | `Tx::subif_start_xmit_8023` |
| `ieee80211_monitor_start_xmit()` | netdev xmit (radiotap injection) | `Tx::monitor_start_xmit` |
| `ieee80211_tx_pending()` (tasklet) | per-restart: drain pending[] queues | `Tx::pending_tasklet` |
| `ieee80211_tx_skb_tid()` | per-stack frame: in-kernel send | `Tx::skb_tid` |
| `ieee80211_xmit_fast_finish()` | per-fast-xmit: late-finish for HW encap | `Tx::xmit_fast_finish` |

## Compatibility contract

REQ-1: struct ieee80211_tx_data:
- skb: per-frame primary skb (may be split into skbs head during fragmentation).
- skbs: per-frame fragments queue (`sk_buff_head`).
- local: per-`struct ieee80211_local` back-pointer.
- sdata: per-`struct ieee80211_sub_if_data` for the originating vif.
- sta: optional per-`struct sta_info` (NULL for mc/injected/mgmt-to-AP).
- key: optional per-`struct ieee80211_key` selected by `h_select_key`.
- rate: per-`struct ieee80211_tx_rate` resolved by `h_rate_ctrl` (legacy path).
- flags: per-`IEEE80211_TX_UNICAST` | `IEEE80211_TX_PS_BUFFERED` | ...

REQ-2: struct ieee80211_tx_info (IEEE80211_SKB_CB):
- flags: per-`IEEE80211_TX_CTL_*` (AMPDU, NO_ACK, INJECTED, ASSIGN_SEQ, AMPDU, DONTFRAG, MORE_FRAMES, FIRST_FRAGMENT, ...) + per-`IEEE80211_TX_INTFL_*` (RETRANSMISSION, OFFCHAN_TX_OK, DONT_ENCRYPT, NEED_TXPROCESSING).
- band: per-`enum nl80211_band`.
- hw_queue: per-AC mapping to driver queue index.
- ack_frame_id / status_data: post-tx ACK tracking.
- control union (pre-tx): vif, hw_key, rates[IEEE80211_TX_MAX_RATES=4], rates_idx, jiffies, enqueue_time, flags (`IEEE80211_TX_CTRL_*`).
- status union (post-tx, alias): rates, ack_signal, ampdu_ack_len, ampdu_len, antenna, tx_time, is_valid_ack_signal, status_driver_data[].

REQ-3: TX handler chain (early, pre-queue):
- /* invoke_tx_handlers_early */
- h_dynamic_ps — driver-no-PS-support fallback: wake AP, queue dynamic-ps-disable work, arm timer.
- h_check_assoc — drop data to !ASSOC STA; drop multicast data when zero mc-if assoc.
- h_ps_buf — multicast → BSS bc_buf; unicast → sta->ps_tx_buf[AC] (returns TX_QUEUED if buffered).
- h_check_control_port_protocol — flag EAPOL: `IEEE80211_TX_CTRL_PORT_CTRL_PROTO`, `USE_MINRATE`, optionally `INTFL_DONT_ENCRYPT`.
- h_select_key — pick PTK[ptk_idx] / link multicast-key / link mgmt-key / sdata default_unicast_key; clears tx->key for cipher-mismatched frame_control; rejects KEY_FLAG_TAINTED non-deauth (TX_DROP).

REQ-4: TX handler chain (late, post-queue):
- /* invoke_tx_handlers_late */
- h_rate_ctrl (skipped if HAS_RATE_CONTROL hw-flag) — call `rate_control_get_rate` (rc80211).
- /* IEEE80211_TX_INTFL_RETRANSMISSION shortcut: tail to skbs, skip rest */
- h_michael_mic_add — TKIP only; append 8-byte MIC (defined in `wpa.c`).
- h_sequence — assign 16-bit `seq_ctrl`: per-sdata global counter or per-(sta,tid) `tx_next_seq`; MLD multicast uses `mld_mcast_seq` (SNS11).
- h_fragment — if `IEEE80211_TX_CTL_DONTFRAG` set or driver-supports `SUPPORTS_TX_FRAG`, skip; else allocate fragment skbs, set MOREFRAGS on all but last, embed SCTL_FRAG; reject AMPDU+fragment (`WARN`).
- h_stats — per-AC byte / packet counters per-(sta, link).
- h_encrypt — dispatch per `tx->key->conf.cipher`: WEP40/104 → `ieee80211_crypto_wep_encrypt`; TKIP → `_tkip_encrypt`; CCMP/CCMP_256 → `_ccmp_encrypt(MIC_LEN)`; AES_CMAC/BIP_CMAC_256 → `_aes_cmac_encrypt`; BIP_GMAC_128/256 → `_aes_gmac_encrypt`; GCMP/GCMP_256 → `_gcmp_encrypt`. Default → TX_DROP.
- h_calculate_duration (skipped if HAS_RATE_CONTROL) — for each fragment, set `hdr->duration_id = ieee80211_duration(tx, skb, group_addr, next_len)`.

REQ-5: ieee80211_tx_prepare(sdata, tx, sta, skb):
- memset(tx); init tx->skbs.
- /* SDATA_AP_VLAN: resolve tx->sta from sdata->u.vlan.sta */
- /* control_port_protocol: use sta_info_get_bss(hdr->addr1) */
- /* unicast non-control: sta_info_get(hdr->addr1); set aggr_check */
- /* If QoS-data, AMPDU_AGGREGATION hw, !TX_AMPDU_SETUP_IN_HW: */
  - tid_tx = rcu_dereference(sta->ampdu_mlme.tid_tx[tid])
  - if missing ∧ aggr_check: ieee80211_aggr_check(sdata, sta, skb); re-load tid_tx.
  - if tid_tx: ieee80211_tx_prep_agg(tx, skb, info, tid_tx, tid) — may return TX_QUEUED.
- /* unicast vs multicast flag */
- /* Fragmentation: set IEEE80211_TX_CTL_DONTFRAG when mc / below frag_threshold / AMPDU */
- info->flags |= IEEE80211_TX_CTL_FIRST_FRAGMENT.
- return TX_CONTINUE.

REQ-6: ieee80211_tx_prep_agg(tx, skb, info, tid_tx, tid):
- /* HT_AGG_STATE_OPERATIONAL → set IEEE80211_TX_CTL_AMPDU; reset agg-timer */
- /* If not yet OPERATIONAL: buffer to tid_tx->pending; return queued=true */
- /* AMPDU buffer overflow: drop oldest */
- /* Per-tid_tx->state STATE_WANT_START: trigger BA-session start workqueue */

REQ-7: ieee80211_queue_skb(local, sdata, sta, skb):
- /* MONITOR: bypass txq */
- /* AP_VLAN: switch sdata to backing AP */
- vif = &sdata->vif.
- txqi = ieee80211_get_txq(local, vif, sta, skb) — pick: sta->sta.txq[tid] for QoS-data, vif->txq_mgmt for buffered MMPDU, vif->txq for non-sta data.
- if !txqi: return false (legacy path → __ieee80211_tx).
- ieee80211_txq_enqueue(local, txqi, skb):
  - /* tid==IEEE80211_NUM_TIDS (mgmt): tail to txqi->frags directly; mark NEED_TXPROCESSING */
  - /* QoS-data: fq_flow_idx, fq_tin_enqueue(fq, tin, flow_idx, skb, fq_skb_free_func) */
- schedule_and_wake_txq(local, txqi): list_add to local->active_txqs[ac] + drv_wake_tx_queue.

REQ-8: ieee80211_tx_dequeue(hw, txq) — driver entry point:
- /* Pre: softirq context (WARN_ON_ONCE) */
- /* AQL gate: ieee80211_txq_airtime_check; bail to NULL if blocked */
- /* hw queue-stop check via local->queue_stop_reasons[q]; if stopped: set IEEE80211_TXQ_DIRTY, NULL */
- /* Pull from txqi->frags first (fragment chain stays together) */
- /* Else fq_tin_dequeue(fq, tin, fq_tin_dequeue_func) — runs CoDel decision */
- Setup `struct ieee80211_tx_data tx` for this skb.
- /* Authorized-port enforcement: drop unicast data to unauthorized STA unless EAPOL / injected */
- /* Re-run h_select_key (key may have rotated while queued) */
- /* IEEE80211_TXQ_AMPDU → set IEEE80211_TX_CTL_AMPDU | _CTL_DONTFRAG */
- /* HW_80211_ENCAP path: only h_rate_ctrl, then encap_out */
- /* IEEE80211_TX_CTRL_FAST_XMIT → ieee80211_xmit_fast_finish */
- /* Else: invoke_tx_handlers_late; dequeue tx.skbs head; splice remaining fragments back to txqi->frags */
- /* skb_has_frag_list ∧ !TX_FRAG_LIST hw: skb_linearize */
- /* Monitor / AP_VLAN sdata fixup */
- /* AQL airtime estimate: ieee80211_calc_expected_tx_airtime; ieee80211_sta_update_pending_airtime */
- return skb.

REQ-9: ieee80211_next_txq(hw, ac) — airtime DRR:
- /* spin_lock(active_txq_lock[ac]) */
- /* schedule_round[ac] gates one pass */
- /* For each txqi on active_txqs[ac]: */
  - aql_check = ieee80211_txq_airtime_check(hw, &txqi->txq).
  - deficit = ieee80211_sta_deficit(sta, ac) = air_info->deficit - aql_tx_pending.
  - if deficit<0: deficit += airtime_weight; move to list tail; continue.
  - if !aql_check: move to tail; continue.
- list_del_init(&txqi->schedule_order); txqi->schedule_round = local->schedule_round[ac].
- return &txqi->txq.

REQ-10: AQL (Airtime Queue Limits):
- /* Per-NL80211_EXT_FEATURE_AQL feature */
- aql_tx_pending: per-(sta, ac) outstanding airtime µs (atomic).
- aql_total_pending: global cap.
- low/high thresholds per-sta; if total > high: block dequeue.

REQ-11: __ieee80211_tx(local, skbs, sta, txpending):
- ieee80211_tx_frags(local, vif, sta, skbs, txpending):
  - For each skb in skbs:
    - q = info->hw_queue.
    - If local->queue_stop_reasons[q] ∨ (!txpending ∧ pending[q] non-empty):
      - if OFFCHAN ∧ stop_reason != OFFCHANNEL only: purge; return.
      - else: splice to local->pending[q]; return false.
    - control.sta = sta ? &sta->sta : NULL.
    - drv_tx(local, &control, skb).
- Returns true if accepted by driver path, false if queued to pending[].

REQ-12: ieee80211_tx_pending tasklet:
- Loops over hw->queues:
  - For each skb on local->pending[q]: ieee80211_tx_pending_skb(local, skb) → re-invoke __ieee80211_tx with txpending=true.

REQ-13: A-MSDU aggregation:
- ieee80211_amsdu_aggregate (called from fast-xmit path): coalesce multiple MSDUs into a single A-MSDU subframe list before final encryption.
- Per-sta `max_amsdu_subframes`, `max_amsdu_len`; per-`IEEE80211_HW_TX_AMSDU` driver opt-in.

REQ-14: Per-AC mapping:
- skb_get_queue_mapping(skb) returns 0..3 (BE=2, BK=3, VI=1, VO=0) per Linux convention with mac80211 inversion.
- ieee80211_ac_from_tid(tid) maps TID 0..7 to AC per WMM.

REQ-15: TXQ teardown:
- ieee80211_txq_purge(local, txqi): fq_tin_reset; purge frags; list_del schedule_order.
- ieee80211_txq_remove_vlan: filter fq entries by control.vif on AP_VLAN remove.

REQ-16: Per-mgmt-frame path:
- ieee80211_tx_skb_tid(sdata, skb, tid, link_id): in-kernel originator (probe-req, action, etc.); sets info, queues via ieee80211_tx.
- ieee80211_monitor_start_xmit: radiotap injection; bypass most handlers.

REQ-17: Fast-xmit:
- Per-sta `fast_xmit` cache (built by `ieee80211_check_fast_xmit`) holds pre-baked 802.11 header template + key for the common-case unicast data path.
- `ieee80211_xmit_fast` consumes the cache; `ieee80211_xmit_fast_finish` runs after dequeue.

REQ-18: ieee80211_tx() top-level:
- ieee80211_tx_prepare → may TX_DROP / TX_QUEUED.
- info->hw_queue set unless TX_CTL_TX_OFFCHAN + QUEUE_CONTROL hw.
- invoke_tx_handlers_early → drop / queued / continue.
- ieee80211_queue_skb → if it returned true: txq path; done.
- Else legacy: invoke_tx_handlers_late + __ieee80211_tx.

## Acceptance Criteria

- [ ] AC-1: Data frame to associated STA: walks dynamic_ps → check_assoc → ps_buf → check_control_port → select_key → rate_ctrl → sequence → fragment → stats → encrypt → calculate_duration → drv_tx.
- [ ] AC-2: EAPOL frame: `IEEE80211_TX_CTRL_PORT_CTRL_PROTO` + `USE_MINRATE` set; passes h_check_assoc on !ASSOC STA via control-port carve-out.
- [ ] AC-3: Data to !ASSOC unicast STA: h_check_assoc → TX_DROP, `tx_handlers_drop_not_assoc++`.
- [ ] AC-4: Multicast data with STAs in PS: h_multicast_ps_buf queues to `bss->ps.bc_buf`, sets `IEEE80211_TX_CTL_SEND_AFTER_DTIM`, returns TX_QUEUED.
- [ ] AC-5: Unicast data to PS-STA: h_unicast_ps_buf queues to `sta->ps_tx_buf[AC]`, recalcs TIM, returns TX_QUEUED.
- [ ] AC-6: CCMP-keyed unicast data: h_select_key picks PTK; h_encrypt dispatches `ieee80211_crypto_ccmp_encrypt(MIC_LEN=8)`.
- [ ] AC-7: KEY_FLAG_TAINTED non-deauth frame, non-control-port protocol: h_select_key → TX_DROP.
- [ ] AC-8: Frame above frag_threshold, no SUPPORTS_TX_FRAG hw, no DONTFRAG, no AMPDU: h_fragment splits into N skbs with MOREFRAGS / SCTL_FRAG indices.
- [ ] AC-9: AMPDU + fragmentation requested: `WARN_ON(IEEE80211_TX_CTL_AMPDU)` in h_fragment → TX_DROP.
- [ ] AC-10: QoS-data with HT_AGG_STATE_OPERATIONAL: ieee80211_tx_prep_agg sets `IEEE80211_TX_CTL_AMPDU | _DONTFRAG`.
- [ ] AC-11: ieee80211_tx_dequeue called outside softirq: WARN_ON_ONCE.
- [ ] AC-12: AQL feature enabled + sta over high threshold: ieee80211_txq_airtime_check → false → dequeue returns NULL.
- [ ] AC-13: Unauthorized STA, unicast data, non-EAPOL, non-injected: dequeue drops + `tx_handlers_drop_unauth_port++`.
- [ ] AC-14: Driver queue stopped + non-txpending: tx_frags splices to `local->pending[q]`; tx_pending tasklet drains on restart.
- [ ] AC-15: ieee80211_next_txq with negative deficit: replenishes by `airtime_weight`, moves to tail.

## Architecture

```
struct TxData {
  skb: Option<SkbPtr>,
  skbs: SkbQueue,
  local: *Local,
  sdata: *SubIfData,
  sta: Option<*StaInfo>,
  key: Option<*Key>,
  rate: TxRate,
  flags: u32,                              // IEEE80211_TX_UNICAST | _PS_BUFFERED | ...
}

struct TxInfo {                            // shared layout with kernel ieee80211_tx_info
  flags: u32,                              // IEEE80211_TX_CTL_*
  band: NL80211Band,
  hw_queue: u8,
  ack_frame_id: u16,
  control: TxInfoControl,                  // pre-tx (vif, hw_key, rates[4], enqueue_time, flags)
  // ... status union aliased after dispatch
}

struct TxqInfo {
  txq: Txq,                                // public part: { vif, sta, tid, ac, drv_priv[] }
  tin: FqTin,                              // FQ-CoDel tin
  def_cvars: CodelVars,
  cstats: CodelStats,
  frags: SkbQueue,                         // fragments + mgmt frames stay together here
  schedule_order: ListNode,                // on local->active_txqs[ac]
  schedule_round: u32,
  flags: AtomicU64,                        // IEEE80211_TXQ_STOP | _AMPDU | _DIRTY
}
```

`Tx::tx(sdata, sta, skb, txpending) -> bool`:
1. /* sub-10 byte skb sanity */
2. res = Tx::prepare(sdata, &tx, sta, skb).
3. if res == TX_DROP: free, sdata.tx_handlers_drop++; return true.
4. if res == TX_QUEUED: return true.
5. /* Set hw_queue unless OFFCHAN + QUEUE_CONTROL */
6. if Tx::invoke_handlers_early(&tx) != 0: return true.
7. if Tx::queue_skb(local, sdata, tx.sta, tx.skb): return true.   // entered txq scheduler
8. /* Legacy direct path (no txq) */
9. if Tx::invoke_handlers_late(&tx) != 0: return true.
10. return Tx::dispatch(local, &tx.skbs, tx.sta, txpending).

`Tx::invoke_handlers_early(tx) -> i32`:
1. CALL_TXH(h_dynamic_ps).
2. CALL_TXH(h_check_assoc).
3. CALL_TXH(h_ps_buf).
4. CALL_TXH(h_check_control_port_protocol).
5. CALL_TXH(h_select_key).
6. /* TX_DROP → free + sdata.tx_handlers_drop++ → -1.
    TX_QUEUED → tx_handlers_queued++ → -1. CONTINUE → 0. */

`Tx::invoke_handlers_late(tx) -> i32`:
1. if !HAS_RATE_CONTROL: CALL_TXH(h_rate_ctrl).
2. if info.flags & IEEE80211_TX_INTFL_RETRANSMISSION:
   - skb_queue_tail(&tx.skbs, tx.skb); tx.skb = None; goto done.
3. CALL_TXH(h_michael_mic_add).
4. CALL_TXH(h_sequence).
5. CALL_TXH(h_fragment).
6. CALL_TXH(h_stats).
7. CALL_TXH(h_encrypt).
8. if !HAS_RATE_CONTROL: CALL_TXH(h_calculate_duration).
9. /* Same TX_DROP / TX_QUEUED handling. */

`Tx::txq_dequeue(hw, txq) -> Option<SkbPtr>`:
1. WARN_ON_ONCE(softirq_count() == 0).
2. if !Tx::txq_airtime_check(hw, txq): return None.
3. loop /* begin: */
4.   spin_lock(queue_stop_reason_lock); q_stopped = queue_stop_reasons[q]; spin_unlock.
5.   if q_stopped: set IEEE80211_TXQ_DIRTY; return None.
6.   spin_lock(fq.lock).
7.   /* fragment chain first */
8.   skb = __skb_dequeue(&txqi.frags).
9.   if skb ∧ !(control.flags & INTCFL_NEED_TXPROCESSING): goto out.
10.  if !skb:
     - if IEEE80211_TXQ_STOP: goto out.
     - skb = fq_tin_dequeue(&fq, &txqi.tin, fq_tin_dequeue_func).
11.  if !skb: goto out.
12.  spin_unlock(fq.lock).
13.  /* Build TxData; auth-port enforcement */
14.  if drop_unauthorized(tx, info, hdr): free; continue.
15.  info.control.hw_key = None; r = Tx::h_select_key(&tx).
16.  if r != CONTINUE: free; continue.
17.  if IEEE80211_TXQ_AMPDU: info.flags |= IEEE80211_TX_CTL_AMPDU | _DONTFRAG.
18.  if HW_80211_ENCAP: optional h_rate_ctrl; goto encap_out.
19.  if IEEE80211_TX_CTRL_FAST_XMIT: Tx::xmit_fast_finish; bail or continue.
20.  else: if Tx::invoke_handlers_late(&tx) != 0: continue.
21.  skb = __skb_dequeue(&tx.skbs); splice remaining frags back to txqi.frags.
22.  if skb_has_frag_list ∧ !TX_FRAG_LIST: skb_linearize or drop.
23.  /* sdata fixup: MONITOR / AP_VLAN */
24.  encap_out: info.control.vif = vif.
25.  /* AQL: ieee80211_calc_expected_tx_airtime + sta_update_pending_airtime(+) */
26.  return Some(skb).
27. out: spin_unlock(fq.lock); return skb.

`Tx::next_txq(hw, ac) -> Option<*Txq>`:
1. spin_lock(active_txq_lock[ac]).
2. if !schedule_round[ac]: out.
3. loop /* begin: */
4.   txqi = list_first_entry_or_null(&active_txqs[ac]).
5.   if !txqi: out.
6.   if txqi == head ∧ !found_eligible: out.
7.   if !head: head = txqi.
8.   if txqi.txq.sta:
     - aql = Tx::txq_airtime_check(hw, &txqi.txq).
     - deficit = sta.airtime[ac].deficit - sta.airtime[ac].aql_tx_pending.
     - if aql: found_eligible = true.
     - if deficit < 0: sta.airtime[ac].deficit += sta.airtime_weight.
     - if deficit < 0 ∨ !aql: list_move_tail(&txqi.schedule_order, &active_txqs[ac]); continue.
9.   if txqi.schedule_round == schedule_round[ac]: out.
10.  list_del_init(&txqi.schedule_order).
11.  txqi.schedule_round = schedule_round[ac].
12.  ret = &txqi.txq.
13. out: spin_unlock(active_txq_lock[ac]); return ret.

`Tx::h_fragment(tx) -> Result`:
1. __skb_queue_tail(&tx.skbs, tx.skb); tx.skb = None.
2. if IEEE80211_TX_CTL_DONTFRAG: return CONTINUE.
3. if SUPPORTS_TX_FRAG hw: return CONTINUE.
4. WARN_ON(IEEE80211_TX_CTL_AMPDU): return DROP.
5. hdrlen = ieee80211_hdrlen(fc).
6. WARN_ON(skb.len + FCS_LEN <= frag_threshold): return DROP.
7. ieee80211_fragment(tx, skb, hdrlen, frag_threshold) — allocates per-fragment skbs.
8. Walk tx.skbs:
   - All but last: hdr.frame_control |= MOREFRAGS; clear rates[1..3]; clear RATE_CTRL_PROBE.
   - Last: hdr.frame_control &= ~MOREFRAGS.
   - hdr.seq_ctrl |= (fragnum & IEEE80211_SCTL_FRAG).
9. return CONTINUE.

`Tx::h_encrypt(tx) -> Result`:
1. if !tx.key: return CONTINUE.
2. match tx.key.conf.cipher:
   - WEP40/104 → crypto_wep_encrypt(tx).
   - TKIP → crypto_tkip_encrypt(tx).
   - CCMP / CCMP_256 → crypto_ccmp_encrypt(tx, MIC_LEN).
   - AES_CMAC / BIP_CMAC_256 → crypto_aes_cmac_encrypt(tx, MIC_LEN).
   - BIP_GMAC_128/256 → crypto_aes_gmac_encrypt(tx).
   - GCMP / GCMP_256 → crypto_gcmp_encrypt(tx).
   - _ → return DROP.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tx_handlers_terminate` | INVARIANT | per-invoke_tx_handlers_{early,late}: chain terminates with CONTINUE | DROP | QUEUED. |
| `tx_data_skbs_invariant` | INVARIANT | per-h_fragment: tx.skb == NULL ⟹ tx.skbs non-empty. |
| `select_key_taint_drop` | INVARIANT | per-h_select_key: KEY_FLAG_TAINTED ∧ !deauth ∧ !control-port ⟹ TX_DROP. |
| `fragment_excludes_ampdu` | INVARIANT | per-h_fragment: IEEE80211_TX_CTL_AMPDU ⟹ DROP. |
| `dequeue_softirq_only` | INVARIANT | per-ieee80211_tx_dequeue: softirq_count() > 0. |
| `aql_gate_obeyed` | INVARIANT | per-tx_dequeue: !airtime_check ⟹ return NULL. |
| `unauth_port_drop` | INVARIANT | per-tx_dequeue: !INJECTED ∧ data ∧ !MESH ∧ !OCB ∧ !mc ∧ !AUTHORIZED ∧ !(PORT_CTRL ∧ our-src) ⟹ drop. |
| `frags_stay_together` | INVARIANT | per-tx_dequeue: txqi.frags head pulled before fq_tin_dequeue. |
| `seqno_assignment` | INVARIANT | per-h_sequence: non-QoS|mc-QoS ⟹ ASSIGN_SEQ + global counter; uc-QoS ⟹ per-(sta,tid) counter. |

### Layer 2: TLA+

`net/mac80211/tx.tla`:
- Per-skb lifecycle states: ENQUEUED → DEQUEUED → DRIVER → ACKED|FAILED.
- Per-AC airtime DRR state: deficit, aql_tx_pending, weight.
- Per-PS buffer model: bc_buf, ps_tx_buf[AC].
- Properties:
  - `safety_no_drop_after_encrypt` — per-frame: if cipher applied, frame either reaches drv_tx or is freed via free_txskb (no leak).
  - `safety_seqno_monotonic_per_sta_tid` — per-(sta, tid): seq_ctrl monotonic mod 2^12.
  - `safety_handler_order_fixed` — per-tx: handlers run in declared order; no skip-after-DROP.
  - `safety_amsdu_excludes_fragment` — per-frame: IEEE80211_TX_CTL_AMPDU never coexists with IEEE80211_TX_CTL_MORE_FRAMES.
  - `liveness_pending_drains_on_wake` — per-q: queue restart wakes tx_pending tasklet; pending[q] empties.
  - `liveness_DRR_fair` — per-AC: every backlogged STA dequeued within bounded rounds.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Tx::prepare` post: tx.local == sdata.local ∧ tx.skbs initialized | `Tx::prepare` |
| `Tx::h_select_key` post: tx.key = NULL ∨ tx.key.conf.cipher matches frame_control class | `Tx::h_select_key` |
| `Tx::h_fragment` post: every skb in tx.skbs has correct MOREFRAGS / SCTL_FRAG | `Tx::h_fragment` |
| `Tx::h_encrypt` post: tx.key != NULL ⟹ skb has cipher header + (MIC | ICV) appended | `Tx::h_encrypt` |
| `Tx::queue_skb` post: ret == true ⟹ txqi non-NULL ∧ skb on txqi (frags or fq_flow) | `Tx::queue_skb` |
| `Tx::txq_dequeue` post: ret skb ⟹ info.control.vif set ∧ (AQL feature ⟹ pending_airtime updated) | `Tx::txq_dequeue` |
| `Tx::next_txq` post: ret txq ⟹ removed from active_txqs[ac] ∧ schedule_round bumped | `Tx::next_txq` |

### Layer 4: Verus/Creusot functional

`Per-tx skb: tx_prepare → handlers_early → queue_skb (fq_tin or frags) → txq_dequeue → handlers_late → drv_tx → status` semantic equivalence: per-IEEE 802.11-2020 §10 (frame exchange sequences) + per-`Documentation/networking/mac80211-injection.rst` + per-FQ-CoDel RFC 8290 for the queueing discipline.

## Hardening

(Inherits row-1 features from `net/mac80211/00-overview.md` § Hardening.)

TX-path reinforcement:

- **Per-handler short-circuit on DROP/QUEUED** — defense against per-handler-skip-after-failure UAF (`CALL_TXH` macro discipline preserved).
- **Per-h_check_assoc gating** — defense against per-data-leak to unassociated STAs.
- **Per-control-port carve-out** — defense against EAPOL-blackhole during 4-way handshake.
- **Per-h_select_key taint check** — defense against per-replayed-key reuse on rotation.
- **Per-h_fragment excludes AMPDU** — defense against per-malformed-MPDU NAV poisoning.
- **Per-frags-stay-together in tx_dequeue** — defense against per-out-of-order MPDU fragmentation.
- **Per-tx_dequeue softirq-only WARN** — defense against per-cross-context fq lock inversion.
- **Per-tx_dequeue unauthorized-port drop** — defense against per-pre-PMF data leak.
- **Per-AQL gate** — defense against per-bufferbloat / per-STA airtime monopolization.
- **Per-airtime DRR deficit accounting** — defense against per-greedy-STA starvation.
- **Per-queue_stop_reasons mask** — defense against per-driver-stop spurious tx (CSA, PS, suspend, off-channel).
- **Per-purge_old_ps_buffers on TOTAL_MAX_TX_BUFFER** — defense against per-PS-buffer exhaustion DoS.
- **Per-KEY_FLAG_UPLOADED_TO_HARDWARE consistency** — defense against per-double-encrypt when offloaded.

## Grsecurity/PaX-style Reinforcement

Rationale: The TX handler chain encrypts and queues every outbound frame — a vtable corruption or key-flag race here can leak plaintext for protected SSIDs (catastrophic) or replay stale CCMP/GCMP PNs (key recovery). TX also implements airtime fairness (AQL + DRR) which is a denial-of-service target: a greedy STA (or hostile driver) can starve other STAs unless the deficit accounting is integrity-protected.

Baseline (cross-ref `net/mac80211/00-overview.md` § Hardening):
- **PAX_USERCOPY**: not in fast path (TX is kernel skb path); netlink-control via `nla_put` only.
- **PAX_KERNEXEC**: `ieee80211_tx_handlers[]`, fq_codel ops, AQL ops placed `__ro_after_init`.
- **PAX_RANDKSTACK**: re-randomise on every `ieee80211_xmit` and tx_dequeue softirq entry.
- **PAX_REFCOUNT**: per-skb-clone + per-sta tx refs saturating.
- **PAX_MEMORY_SANITIZE**: freed TX skbs + per-PS-buffer slots zero-filled (defends PS-buffer leak after STA disassoc).
- **PAX_UDEREF**: no direct user-deref in TX path.
- **PAX_RAP / kCFI**: `ieee80211_tx_handler` chain + driver `->tx` + `->wake_tx_queue` kCFI-tagged.
- **GRKERNSEC_HIDESYM**: skb + sta_info + key pointers never logged (only %pK).
- **GRKERNSEC_DMESG**: queue-stop and PS-buffer-purge logs ratelimited.

mac80211-tx-specific reinforcement:
- **ieee80211_local PAX_REFCOUNT** — `local->q_stop_reasons[]` and per-sta `sta->tx_stats` use saturating Refcount.
- **Key MEMORY_SANITIZE strict** — keys removed from TX path memzero_explicit's material before slab-free; PN counter zeroed.
- **sta_info refcount** saturating across the TX chain; defends per-sta refcount imbalance during disconnect race.
- **AQL bound strict** — refuse driver-reported airtime > GRSEC_AQL_PERFRAME_MAX_NS; defends driver-bug-induced unbounded airtime credit.
- **DRR deficit integer overflow guard** — deficit accumulator capped at GRSEC_DRR_DEFICIT_MAX (saturating, not wrap).
- **TX queue length cap per STA** — refuse > TOTAL_MAX_TX_BUFFER (already row-2) — extended with GR audit-log + per-STA penalty.
- **Pre-PMF auth-port unauthorized drop strict** — already row-2; extended with audit-log on attempted pre-4WHS data frame.
- **KEY_FLAG_UPLOADED_TO_HARDWARE consistency** (already row-2) — extended with GR audit-log on offload/sw mismatch.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TKIP / CCMP / GCMP / GMAC / WEP cipher kernels (covered in `net/mac80211/wpa.md` and `net/mac80211/aead_api.md`).
- A-MPDU BA session lifecycle (addba/delba) (covered in `net/mac80211/agg-tx.md`).
- Rate control algorithms (`rc80211_minstrel*`) (covered in `net/mac80211/rate.md`).
- TX status / ACK processing (covered in `net/mac80211/status.md`).
- TXQ scheduler driver hooks (`drv_wake_tx_queue`) and individual driver TX paths.
- HT/VHT/HE/EHT capability negotiation (covered in `net/mac80211/ht.md` / `vht.md` / `he.md`).
- Implementation code.
