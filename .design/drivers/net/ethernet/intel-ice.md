# Tier-3: drivers/net/ethernet/intel/ice/ — Intel E810 (ice) enterprise NIC driver

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/00-overview.md
upstream-paths:
  - drivers/net/ethernet/intel/ice/ice_main.c (~9815 lines)
  - drivers/net/ethernet/intel/ice/ice.h
  - drivers/net/ethernet/intel/ice/ice_lib.c
  - drivers/net/ethernet/intel/ice/ice_base.c
  - drivers/net/ethernet/intel/ice/ice_txrx.c
  - drivers/net/ethernet/intel/ice/ice_txrx_lib.c
  - drivers/net/ethernet/intel/ice/ice_common.c
  - drivers/net/ethernet/intel/ice/ice_controlq.c
  - drivers/net/ethernet/intel/ice/ice_sriov.c
  - drivers/net/ethernet/intel/ice/ice_vf_lib.c
  - drivers/net/ethernet/intel/ice/ice_xsk.c (AF_XDP)
  - drivers/net/ethernet/intel/ice/ice_xsk.h
  - drivers/net/ethernet/intel/ice/ice_dcb.c / ice_dcb_lib.c / ice_dcb_nl.c
  - drivers/net/ethernet/intel/ice/ice_ddp.c (Dynamic Device Personalization)
  - drivers/net/ethernet/intel/ice/ice_fdir.c / ice_ethtool_fdir.c
  - drivers/net/ethernet/intel/ice/ice_flow.c / ice_flex_pipe.c
  - drivers/net/ethernet/intel/ice/ice_arfs.c (aRFS)
  - drivers/net/ethernet/intel/ice/ice_idc.c (RDMA core IDC)
  - drivers/net/ethernet/intel/ice/ice_ptp.c (PTP / PHC)
  - drivers/net/ethernet/intel/ice/ice_irq.c
  - drivers/net/ethernet/intel/ice/ice_sched.c (per-Tx scheduler tree)
  - drivers/net/ethernet/intel/ice/ice_switch.c
  - drivers/net/ethernet/intel/ice/ice_tc_lib.c
  - drivers/net/ethernet/intel/ice/ice_eswitch.c
  - drivers/net/ethernet/intel/ice/ice_lag.c
  - drivers/net/ethernet/intel/ice/ice_dpll.c (SyncE/PTP-DPLL)
  - drivers/net/ethernet/intel/ice/ice_gnss.c
  - drivers/net/ethernet/intel/ice/ice_devids.h
  - drivers/net/ethernet/intel/ice/ice_type.h
  - drivers/net/ethernet/intel/ice/ice_adminq_cmd.h
  - drivers/net/ethernet/intel/ice/ice_hw_autogen.h (HW register map)
  - Documentation/networking/device_drivers/ethernet/intel/ice.rst
-->

## Summary

The **ice** driver supports Intel's E810 / E822 / E823 / E825 / E830 family of 100 GbE Ethernet controllers — the canonical "large modern PCIe NIC" in Linux. It is built around a **PF (Physical Function) / VSI (Virtual Station Interface)** abstraction: `struct ice_pf` owns one PCI device + one admin queue + one mailbox queue + one sideband queue + N VSIs; `struct ice_vsi` owns a netdev + a set of queue-pairs (Tx ring + Rx ring) + a set of `struct ice_q_vector` (NAPI poll context + MSI-X vector); `struct ice_q_vector` owns one Rx ring + zero-or-more Tx rings + one NAPI struct + one IRQ. The driver implements: PCIe enumeration via `ice_probe` / `ice_remove`; FW interaction via the **AdminQ** (control queue) for VSI / switch / RSS / Tx-scheduler / FDIR / DDP commands; per-VSI Tx queue tree managed through the firmware-resident **scheduler** (`ice_sched.c`); per-PF SR-IOV with up to 256 VFs and per-VF mailbox; XDP-DRV native paths with up to one XDP-Tx queue per NAPI vector; AF_XDP zero-copy via `ice_xsk.c`; **DDP (Dynamic Device Personalization)** firmware-loaded protocol-parser package `intel/ice/ddp/ice.pkg`; FDIR (Flow Director) hashing + perfect-match filters via `ice_ethtool_fdir.c`; aRFS via `ice_arfs.c`; DCB (Data Center Bridging) ETS / PFC / IEEE-LLDP via `ice_dcb*.c`; ETF (Earliest TxTime First) qdisc offload via `ice_offload_txtime`; RDMA / iWARP+RoCE via the IDC (Inter-Driver Communication) auxiliary bus to `irdma`; PTP / SyncE via `ice_ptp.c` + `ice_dpll.c`; switchdev / eSwitch + representor ports via `ice_eswitch.c`; LAG (Link Aggregation) team-mode via `ice_lag.c`. Critical for: deterministic 100 GbE forwarding, container / cloud NIC offload, AF_XDP DPDK-style userspace networking, RDMA NVMe-oF / SMB-Direct, time-sensitive networking (TSN), and SR-IOV multi-tenant deployments.

This Tier-3 covers `drivers/net/ethernet/intel/ice/ice_main.c` (~9815 lines) and the `ice/` subdir (~113 files, ~97 kSLOC) as the canonical large-PCIe-NIC reference.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ice_pf` | per-PCIe-function context | `drivers::intel::ice::Pf` |
| `struct ice_vsi` | per-VSI (netdev + queues) | `Vsi` |
| `struct ice_q_vector` | per-NAPI + MSI-X | `QVector` |
| `struct ice_tx_ring` / `ice_rx_ring` | per-queue descriptor ring | `TxRing` / `RxRing` |
| `struct ice_hw` | per-HW register-access + caps | `Hw` |
| `struct ice_adapter` | per-pdev sibling-PF coordinator | `Adapter` |
| `struct pci_driver ice_driver` | per-PCIe registration | `ICE_DRIVER` |
| `ice_module_init()` / `_exit()` | per-load | `Module::init` / `exit` |
| `ice_probe()` | per-PCIe device probe | `Pf::probe` |
| `ice_remove()` | per-PCIe device unbind | `Pf::remove` |
| `ice_shutdown()` | per-PCIe shutdown | `Pf::shutdown` |
| `ice_init_hw()` / `ice_deinit_hw()` | per-AdminQ + caps init | `Hw::init` / `deinit` |
| `ice_init_dev()` / `ice_deinit_dev()` | per-pre-VSI device init | `Pf::init_dev` / `deinit_dev` |
| `ice_init()` / `ice_deinit()` | per-VSI-table + link init | `Pf::init` / `deinit` |
| `ice_load()` / `ice_unload()` | per-netdev + features + RDMA | `Pf::load` / `unload` |
| `ice_alloc_vsis()` | per-vsi-array alloc | `Pf::alloc_vsis` |
| `ice_init_pf_sw()` | per-default-switch + main-VSI | `Pf::init_pf_sw` |
| `ice_pf_vsi_setup()` / `ice_vsi_setup()` | per-VSI alloc + cfg | `Vsi::setup` |
| `ice_vsi_release()` | per-VSI tear-down | `Vsi::release` |
| `ice_register_netdev()` / `_unregister_netdev` | per-netdev (un)register | `Vsi::register_netdev` / `unregister` |
| `ice_cfg_netdev()` / `ice_decfg_netdev()` | per-netdev allocate + features | `Vsi::cfg_netdev` / `decfg_netdev` |
| `ice_napi_add()` / `_napi_del` | per-vector napi (de)register | `QVector::napi_add` / `del` |
| `ice_napi_poll()` | per-NAPI Tx + Rx clean | `QVector::napi_poll` |
| `ice_msix_clean_rings()` | per-vector MSI-X handler | `QVector::msix_irq` |
| `ice_misc_intr()` / `ice_misc_intr_thread_fn()` | per-OICR (other-interrupt-cause-register) handler + threaded fn | `Pf::misc_intr` / `misc_intr_thread` |
| `ice_ll_ts_intr()` | per-low-latency Tx timestamp IRQ | `Pf::ll_ts_intr` |
| `ice_req_irq_msix_misc()` | per-OICR + LL-TS IRQ request | `Pf::req_irq_msix_misc` |
| `ice_free_irq_msix_misc()` | per-OICR IRQ release | `Pf::free_irq_msix_misc` |
| `ice_ena_misc_vector()` / `ice_ena_ctrlq_interrupts()` / `ice_dis_ctrlq_interrupts()` | per-OICR + AdminQ + MBX + SB IRQ enable/disable | `Pf::ena_misc_vector` / `ena_ctrlq_interrupts` / `dis_ctrlq_interrupts` |
| `ice_init_interrupt_scheme()` / `_clear_interrupt_scheme()` | per-MSI-X vector alloc | `Pf::init_interrupt_scheme` / `clear` |
| `ice_alloc_irq()` / `ice_free_irq()` | per-vector lend from irq_tracker | `Pf::alloc_irq` / `free_irq` |
| `ice_vsi_alloc_q_vectors()` / `ice_vsi_free_q_vectors()` | per-VSI q_vector array | `Vsi::alloc_q_vectors` / `free` |
| `ice_vsi_map_rings_to_vectors()` | per-VSI ring-to-vector binding | `Vsi::map_rings_to_vectors` |
| `ice_vsi_cfg_txqs()` / `ice_vsi_cfg_rxqs()` | per-VSI HW queue programming | `Vsi::cfg_txqs` / `cfg_rxqs` |
| `ice_vsi_open()` / `ice_vsi_close()` | per-VSI bring up / tear down | `Vsi::open` / `close` |
| `ice_open()` / `ice_stop()` | per-netdev ndo_open / ndo_stop | `Vsi::open_netdev` / `stop_netdev` |
| `ice_up()` / `ice_down()` | per-VSI fast restart | `Vsi::up` / `down` |
| `ice_start_xmit()` | per-skb ndo_start_xmit | `TxRing::start_xmit` |
| `ice_clean_tx_irq()` | per-NAPI Tx completion drain | `TxRing::clean` |
| `ice_clean_rx_irq()` | per-NAPI Rx descriptor drain | `RxRing::clean` |
| `ice_xdp()` / `ice_xdp_setup_prog()` | per-XDP_SETUP_PROG + XDP_SETUP_XSK_POOL | `Vsi::xdp` / `xdp_setup_prog` |
| `ice_xdp_xmit()` | per-XDP_REDIRECT TX path | `Vsi::xdp_xmit` |
| `ice_prepare_xdp_rings()` / `ice_destroy_xdp_rings()` | per-XDP-Tx-ring alloc/free | `Vsi::prepare_xdp_rings` / `destroy` |
| `ice_xsk_pool_setup()` / `ice_xsk_wakeup()` | per-AF_XDP pool bind + wakeup | `Xsk::pool_setup` / `wakeup` |
| `ice_xmit_zc()` / `ice_clean_rx_irq_zc()` | per-AF_XDP zerocopy Tx/Rx | `Xsk::xmit_zc` / `clean_rx_zc` |
| `ice_sriov_configure()` | per-PCI sysfs num_vfs | `Sriov::configure` |
| `ice_free_vfs()` / `ice_alloc_vfs()` | per-VF lifecycle | `Sriov::free_vfs` / `alloc_vfs` |
| `ice_set_vf_mac()` / `ice_set_vf_link_state()` / `ice_get_vf_cfg()` / `ice_set_vf_trust()` / `ice_set_vf_spoofchk()` / `ice_set_vf_port_vlan()` / `ice_set_vf_bw()` | per-VF NDO ops | `Sriov::set_*` / `get_*` |
| `ice_vc_notify_reset()` / `ice_vc_process_vf_msg()` | per-VF mailbox virtchnl | `Sriov::vc_*` |
| `ice_init_pf_dcb()` / `ice_cfg_lldp_mib_change()` / `ice_dcbnl_setup()` | per-DCB init + LLDP + netlink | `Dcb::init_pf` / `cfg_lldp_mib_change` / `dcbnl_setup` |
| `ice_init_fdir()` / `ice_deinit_fdir()` | per-Flow-Director init | `Fdir::init` / `deinit` |
| `ice_set_rss_lut()` / `ice_set_rss_key()` / `ice_set_rss_hfunc()` | per-RSS LUT + key + hash-fn | `Rss::set_lut` / `set_key` / `set_hfunc` |
| `ice_init_rdma()` / `ice_deinit_rdma()` | per-RDMA-IDC aux-dev plug | `Rdma::init` / `deinit` |
| `ice_rdma_finalize_setup()` | per-RDMA aux-dev registration | `Rdma::finalize_setup` |
| `ice_load_pkg()` (via `ice_init_hw`) | per-DDP firmware load | `Ddp::load_pkg` |
| `ice_log_pkg_init()` | per-DDP-state log | `Ddp::log_pkg_init` |
| `ice_request_fw()` | per-DDP firmware request | `Ddp::request_fw` |
| `ice_offload_txtime()` | per-ETF qdisc TC_SETUP_QDISC_ETF | `Tx::offload_txtime` |
| `ice_cfg_txtime()` | per-Tx-ring earliest-TxTime enable | `TxRing::cfg_txtime` |
| `ice_setup_tc()` | per-TC_SETUP_* dispatch | `Tc::setup_tc` |
| `ice_setup_tc_mqprio_qdisc()` | per-MQPRIO offload | `Tc::setup_mqprio_qdisc` |
| `ice_ptp_init()` / `ice_ptp_release()` / `ice_ptp_ts_irq()` / `ice_ptp_process_ts()` | per-PTP PHC + Tx-timestamp | `Ptp::init` / `release` / `ts_irq` / `process_ts` |
| `ice_dpll_init()` / `_deinit` | per-SyncE/PTP DPLL | `Dpll::init` / `deinit` |
| `ice_gnss_init()` / `_exit` | per-GNSS UART | `Gnss::init` / `exit` |
| `ice_init_lag()` / `_deinit_lag` | per-LAG teaming | `Lag::init` / `deinit` |
| `ice_eswitch_*` family | per-switchdev + representor | `Eswitch::*` |
| `ice_service_task()` / `_schedule()` / `_stop` / `_restart` | per-PF deferred work (link / reset / VFLR / MDD) | `Pf::service_task` / `schedule` / `stop` / `restart` |
| `ice_pci_err_handler` (`ice_pci_err_detected` / `_reset_prepare` / `_reset_done` / `_resume`) | per-AER (PCIe-error) callbacks | `Pf::aer_*` |
| `ice_pm_ops` (`ice_suspend` / `ice_resume`) | per-PM D3 / D0 | `Pf::suspend` / `resume` |
| `ice_pf_reset()` / `ice_do_reset()` / `ice_rebuild()` | per-PFR / CORER / GLOBR / EMPR reset orchestration | `Pf::reset` / `do_reset` / `rebuild` |
| `ice_wq` / `ice_lag_wq` | per-driver workqueues | `ICE_WQ` / `ICE_LAG_WQ` |
| `ice_netdev_ops` / `ice_netdev_safe_mode_ops` | per-net_device_ops vtable | `ICE_NETDEV_OPS` / `ICE_NETDEV_SAFE_MODE_OPS` |
| `ice_xdp_md_ops` | per-xdp_metadata_ops vtable | `ICE_XDP_MD_OPS` |
| `ice_pci_tbl[]` | per-PCI ID match | `ICE_PCI_TBL` |

## Compatibility contract

REQ-1: struct ice_pf (`drivers/net/ethernet/intel/ice/ice.h:556`):
- pdev: per-pci_dev pointer.
- adapter: per-pdev `ice_adapter` (shared between sibling PFs on the same package).
- hw: per-`ice_hw` (FW + register access + caps).
- vsi: per-array (`pf->num_alloc_vsi`) of `ice_vsi *`.
- vsi_stats: per-VSI net-stats accumulators.
- first_sw: per-firmware-default switch element.
- state: DECLARE_BITMAP of `ICE_*` (`ICE_DOWN` / `ICE_SERVICE_DIS` / `ICE_RESET_OICR_RECV` / `ICE_PFR_REQ` / `ICE_CORER_REQ` / `ICE_GLOBR_REQ` / `ICE_VF_RESETS_DISABLED` / `ICE_SUSPENDED` / `ICE_ADMINQ_EVENT_PENDING` / `ICE_MAILBOXQ_EVENT_PENDING` / `ICE_SIDEBANDQ_EVENT_PENDING` / `ICE_MDD_EVENT_PENDING` / `ICE_VFLR_EVENT_PENDING` / `ICE_AUX_ERR_PENDING` / `ICE_NEEDS_RESTART` / ...).
- flags: bitmap of `enum ice_pf_flags` (`ICE_FLAG_FLTR_SYNC` / `ICE_FLAG_RDMA_ENA` / `ICE_FLAG_RSS_ENA` / `ICE_FLAG_SRIOV_ENA` / `ICE_FLAG_DCB_CAPABLE` / `ICE_FLAG_DCB_ENA` / `ICE_FLAG_FD_ENA` / `ICE_FLAG_PTP_SUPPORTED` / `ICE_FLAG_NO_MEDIA` / `ICE_FLAG_GNSS` / `ICE_FLAG_DPLL` / ...).
- oicr_irq, ll_ts_irq: per-`struct msi_map` for OICR + low-latency-TS IRQs.
- msix: per-`struct ice_pf_msix` `{cur, min, max, total, rest}`.
- avail_txqs, avail_rxqs: per-bitmap of available HW queues.
- num_alloc_vsi, num_lan_msix, num_rdma_msix: per-counter.
- serv_tmr: per-PF watchdog timer.
- service_task: per-PF work_struct.
- int_name, int_name_ll_ts: per-IRQ name (printed in /proc/interrupts).
- corer_count, globr_count, empr_count, pfr_count: per-reset-type counters.
- adapter / aux_idx / cdev_info: per-IDC aux-bus state.
- txtime_txqs: per-ETF Tx-queue bitmap.
- eswitch: per-switchdev / representor state.

REQ-2: struct ice_vsi (`drivers/net/ethernet/intel/ice/ice.h:334`):
- netdev: per-VSI net_device.
- vsw: per-software-switch pointer.
- back: per-PF pointer.
- rx_rings, tx_rings, xdp_rings: per-ring pointer arrays.
- q_vectors: per-NAPI vector array.
- irq_handler: per-VSI IRQ entry (`ice_msix_clean_rings`).
- num_q_vectors, num_txq, num_rxq, num_xdp_txq: per-VSI sizing.
- alloc_txq, alloc_rxq: per-allocated queue counts.
- vsi_num, idx: HW (absolute) + SW (pf->vsi[idx]) handles.
- rss_table_size, rss_size, rss_hfunc, rss_hkey_user, rss_lut_user, rss_lut_type: per-RSS config.
- arfs_fltr_list, arfs_fltr_cntrs, arfs_lock: per-aRFS hash table.
- info: `ice_aqc_vsi_props` (AdminQ-format properties).
- vlan_info, inner_vlan_ops, outer_vlan_ops, num_vlan: per-VLAN.
- tc_cfg, mqprio_qopt, all_numtc, all_enatc, old_numtc, old_ena_tc, tc_map_vsi[], num_chnl_rxq, num_chnl_txq, ch_rss_size: per-TC + MQPRIO + ADQ channel state.
- xdp_prog, xdp_state_lock: per-XDP program + program-swap mutex.
- agg_node: per-aggregator-node back-pointer (`ice_sched`).
- params (struct_group_tagged): port_info, ch, vf|sf, flags, type.
- state: DECLARE_BITMAP of `ICE_VSI_*` (`ICE_VSI_DOWN` / `ICE_VSI_REBUILD_PENDING` / ...).
- type: `ice_vsi_type` (`ICE_VSI_PF` / `ICE_VSI_VF` / `ICE_VSI_CTRL` / `ICE_VSI_CHNL` / `ICE_VSI_LB` / `ICE_VSI_SF`).

REQ-3: struct ice_q_vector (`drivers/net/ethernet/intel/ice/ice.h:468`):
- vsi: per-back-pointer.
- v_idx: per-VSI index.
- reg_idx: per-PF HW vector index.
- num_ring_rx, num_ring_tx: per-vector ring counts.
- wb_on_itr (bit): WB-on-ITR enabled.
- intrl: per-interrupt-rate-limit usec.
- napi: per-napi_struct.
- rx, tx: per-`ice_ring_container` (head + budget + ITR-state).
- ch: per-ADQ channel back-pointer.
- name: per-IRQ name.
- total_events: per-NAPI events counter.
- irq: per-`struct msi_map`.

REQ-4: ice_module_init():
- `pr_info("%s\n", ice_driver_string)` + copyright.
- `ice_adv_lnk_speed_maps_init()`.
- `ice_wq = alloc_workqueue("%s", WQ_UNBOUND, 0, KBUILD_MODNAME)`; if fail return -ENOMEM.
- `ice_lag_wq = alloc_ordered_workqueue("ice_lag_wq", 0)`; unroll on fail.
- `ice_debugfs_init()`.
- `pci_register_driver(&ice_driver)`; unroll on fail.
- `ice_sf_driver_register()` (subfunction aux driver); unroll on fail.
- Return 0.

REQ-5: ice_probe(pdev, ent):
- if pdev->is_virtfn: return -EINVAL.
- if is_kdump_kernel(): pci_save_state + pci_clear_master + pcie_flr + pci_restore_state.
- pcim_enable_device(pdev); pcim_iomap_regions(pdev, BIT(ICE_BAR0), ...).
- pf = ice_allocate_pf(dev); pf->aux_idx = -1.
- dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64)).
- pci_set_master(pdev); pf->pdev = pdev; pci_set_drvdata(pdev, pf).
- set_bit(ICE_DOWN, pf->state); set_bit(ICE_SERVICE_DIS, pf->state).
- hw = &pf->hw; hw->hw_addr = pcim_iomap_table(pdev)[ICE_BAR0]; pci_save_state.
- Populate hw->vendor_id / device_id / revision_id / subsystem_*; bus.device, bus.func.
- ice_set_ctrlq_len(hw).
- pf->msg_enable = netif_msg_init(debug, ICE_DFLT_NETIF_M).
- if ice_is_recovery_mode(hw): return ice_probe_recovery_mode(pf).
- ice_init_hw(hw) — AdminQ init, FW caps, DDP load, scheduler tree.
- ice_init_dev_hw(pf).
- adapter = ice_adapter_get(pdev); pf->adapter = adapter.
- ice_init_dev(pf) — irq scheme, FDIR, MAC.
- ice_init(pf) — alloc_vsis, init_pf_sw, init_link, send_version.
- devl_lock(priv_to_devlink(pf)); ice_load(pf); ice_init_devlink(pf); devl_unlock.
- Reverse-unroll order: ice_unload → ice_deinit → ice_adapter_put → ice_deinit_hw → ice_deinit_dev.

REQ-6: ice_init() → ice_load() sub-stages:
- ice_init_pf(pf) (workqueue, locks, bitmaps, service-task).
- if mac_type == ICE_MAC_E830: pci_enable_ptm(pdev).
- ice_alloc_vsis(pf) (`pf->num_alloc_vsi = pf->hw.func_caps.guar_num_vsi`; capped at `UDP_TUNNEL_NIC_MAX_SHARING_DEVICES`).
- ice_init_pf_sw(pf) — kzalloc(pf->first_sw); set bridge mode (VEB / VEPA); `ice_aq_set_port_params(pi, dvm, NULL)`; `ice_pf_vsi_setup(pf, pf->hw.port_info)` allocates the main VSI.
- ice_init_wakeup(pf) — read PFPM_WUS; print wake reason; clear; device_set_wakeup_enable(dev, false).
- ice_init_link(pf) — `ice_init_link_events`, `ice_init_nvm_phy_type`, `ice_update_link_info`, `ice_init_link_dflt_override`, `ice_init_phy_user_cfg`, `ice_phy_cfg(vsi, link_en)`.
- ice_send_version(pf) — AdminQ `set_drv_ver`.
- ice_verify_cacheline_size(pf).
- if safe-mode: ice_set_safe_mode_vlan_cfg(pf) else pcie_print_link_status.
- clear_bit(ICE_DOWN, ...); clear_bit(ICE_SERVICE_DIS, ...).
- mod_timer(&pf->serv_tmr, round_jiffies(jiffies + pf->serv_tmr_period)).
- ice_load: cfg_netdev → dcbnl_setup → init_mac_fltr → devlink_create_pf_port → register_netdev → tc_indir_block_register → napi_add → init_features → init_rdma → rdma_finalize_setup → service_task_restart → clear ICE_DOWN.

REQ-7: ice_init_features(pf) (post-load):
- if safe-mode: return.
- if `ICE_FLAG_PTP_SUPPORTED`: ice_ptp_init(pf).
- if `ICE_F_GNSS`: ice_gnss_init(pf).
- if `ICE_F_CGU` || `ICE_F_PHY_RCLK`: ice_dpll_init(pf).
- ice_init_fdir(pf) — non-fatal.
- ice_init_pf_dcb(pf, false) — if fails clear `ICE_FLAG_DCB_CAPABLE` + `ICE_FLAG_DCB_ENA`; else `ice_cfg_lldp_mib_change(&pf->hw, true)`.
- ice_init_lag(pf) — non-fatal.
- ice_hwmon_init(pf).

REQ-8: ice_remove(pdev) (`ice_main.c:5358`):
- Wait up to `ICE_MAX_RESET_WAIT` × 100 ms for in-progress reset to drain.
- if recovery-mode: ice_service_task_stop + scoped_guard(devl) ice_deinit_devlink; return.
- if ICE_FLAG_SRIOV_ENA: set_bit(ICE_VF_RESETS_DISABLED) + ice_free_vfs.
- if !safe-mode: ice_remove_arfs.
- devl_lock → ice_dealloc_all_dynamic_ports → ice_deinit_devlink → ice_unload → devl_unlock.
- ice_deinit → ice_vsi_release_all → ice_setup_mc_magic_wake → ice_set_wake → ice_adapter_put → ice_deinit_hw → ice_deinit_dev → ice_aq_cancel_waiting_tasks → set_bit(ICE_DOWN).

REQ-9: ice_req_irq_msix_misc(pf) — OICR + LL-TS interrupt setup:
- Build pf->int_name = "ice-<dev>:misc" and pf->int_name_ll_ts = "ice-<dev>:ll_ts".
- if reset-in-progress: skip request (settings lost during reset; re-enable only).
- irq = ice_alloc_irq(pf, false); pf->oicr_irq = irq.
- devm_request_threaded_irq(dev, pf->oicr_irq.virq, ice_misc_intr, ice_misc_intr_thread_fn, 0, pf->int_name, pf).
- if `ts_dev_info.ts_ll_int_read`: ice_alloc_irq + devm_request_irq(virq, ice_ll_ts_intr, 0, int_name_ll_ts, pf).
- ice_ena_misc_vector(pf).
- ice_ena_ctrlq_interrupts(hw, pf->oicr_irq.index) — AdminQ + Mailbox + Sideband + OICR enable.
- pf_intr_start_offset = rd32(hw, PFINT_ALLOC) & PFINT_ALLOC_FIRST.
- wr32(hw, GLINT_ITR(ICE_RX_ITR, pf->oicr_irq.index), ITR_REG_ALIGN(ICE_ITR_8K) >> ICE_ITR_GRAN_S).
- ice_flush + ice_irq_dynamic_ena(hw, NULL, NULL).

REQ-10: ice_misc_intr(irq, pf) — OICR top half (`ice_main.c:3097`):
- set_bit(ICE_ADMINQ_EVENT_PENDING) + ICE_MAILBOXQ_EVENT_PENDING + ICE_SIDEBANDQ_EVENT_PENDING.
- oicr = rd32(hw, PFINT_OICR); ena_mask = rd32(hw, PFINT_OICR_ENA).
- PFINT_OICR_SWINT_M: pf->sw_int_count++; clear in ena_mask.
- PFINT_OICR_MAL_DETECT_M: set ICE_MDD_EVENT_PENDING.
- PFINT_OICR_VFLR_M: if ICE_VF_RESETS_DISABLED — disable in PFINT_OICR_ENA; else set ICE_VFLR_EVENT_PENDING.
- PFINT_OICR_GRST_M: read GLGEN_RSTAT reset-type; bump corer_count|globr_count|empr_count; if !ICE_RESET_OICR_RECV — set ICE_RESET_OICR_RECV + ICE_CORER_RECV|ICE_GLOBR_RECV|ICE_EMPR_RECV; hw->reset_ongoing = true.
- PFINT_OICR_TSYN_TX_M: ret = ice_ptp_ts_irq(pf).
- PFINT_OICR_TSYN_EVNT_M: read GLTSYN_STAT; if pf is src-tmr-owner: pf->ptp.ext_ts_irq |= ...; ice_ptp_extts_event(pf).
- PFINT_OICR_PE_CRITERR_M | PFINT_OICR_HMC_ERR_M | PFINT_OICR_PE_PUSH_M (ICE_AUX_CRIT_ERR): record in pf->oicr_err_reg, set ICE_AUX_ERR_PENDING.
- residual oicr & ena_mask: if PFINT_OICR_PCI_EXCEPTION_M | PFINT_OICR_ECC_ERR_M — set ICE_PFR_REQ.
- ice_service_task_schedule(pf).
- if ret == IRQ_HANDLED: ice_irq_dynamic_ena(hw, NULL, NULL).
- return ret.

REQ-11: ice_misc_intr_thread_fn(irq, pf) — OICR threaded half (`ice_main.c:3234`):
- if reset-in-progress: skip irq.
- if test_and_clear_bit(ICE_MISC_THREAD_TX_TSTAMP, pf->misc_thread): ice_ptp_process_ts(pf).
- ice_irq_dynamic_ena + ice_flush.
- if ice_ptp_tx_tstamps_pending: wr32(PFINT_OICR, PFINT_OICR_TSYN_TX_M) — re-arm + ice_flush.
- return IRQ_HANDLED.

REQ-12: ice_msix_clean_rings(irq, q_vector) — per-data-path IRQ (`ice_lib.c:502`):
- if !q_vector->tx.tx_ring && !q_vector->rx.rx_ring: return IRQ_HANDLED.
- q_vector->total_events++.
- napi_schedule(&q_vector->napi).
- return IRQ_HANDLED.

REQ-13: ice_napi_poll(napi, budget) — per-NAPI Tx + Rx clean (`ice_txrx.c:1267`):
- for_each_tx_ring(tx_ring, q_vector->tx):
  - xsk_pool = READ_ONCE(tx_ring->xsk_pool).
  - if xsk_pool: wd = ice_xmit_zc(tx_ring, xsk_pool).
  - else if ring_is_xdp(tx_ring): wd = true.
  - else: wd = ice_clean_tx_irq(tx_ring, budget).
  - if !wd: clean_complete = false.
- if budget ≤ 0 (netpoll): return budget.
- budget_per_ring = num_ring_rx > 1 ? max(budget/num_ring_rx, 1) : budget.
- for_each_rx_ring(rx_ring, q_vector->rx):
  - xsk_pool = READ_ONCE(rx_ring->xsk_pool).
  - cleaned = xsk_pool ? ice_clean_rx_irq_zc(rx_ring, xsk_pool, budget_per_ring) : ice_clean_rx_irq(rx_ring, budget_per_ring).
  - work_done += cleaned; if cleaned >= budget_per_ring: clean_complete = false.
- if !clean_complete: ice_set_wb_on_itr(q_vector); return budget.
- if napi_complete_done(napi, work_done): ice_net_dim(q_vector); ice_enable_interrupt(q_vector); else ice_set_wb_on_itr(q_vector).
- return min(work_done, budget - 1).

REQ-14: ice_open(netdev) / ice_open_internal(netdev) (`ice_main.c:9588` / `:9609`):
- if reset-in-progress: -EBUSY.
- if ICE_NEEDS_RESTART: -EIO.
- netif_carrier_off.
- pi = vsi->port_info; ice_update_link_info(pi); ice_check_link_cfg_err.
- if MEDIA_AVAILABLE: clear ICE_FLAG_NO_MEDIA; if !ICE_PHY_INIT_COMPLETE: ice_init_phy_user_cfg; ice_phy_cfg(vsi, true).
- else: set ICE_FLAG_NO_MEDIA; ice_set_link(vsi, false).
- ice_vsi_open(vsi) — alloc Rx buffers, configure rings via AdminQ, request per-vector IRQs (`ice_msix_clean_rings`), enable interrupts, napi_enable, netif_tx_start.

REQ-15: ice_stop(netdev):
- ice_vsi_close(vsi) — netif_carrier_off, netif_tx_disable, napi_disable, disable per-vector IRQs (synchronize_irq + free_irq), `ice_vsi_dis_irq`, AdminQ-stop queues, ice_clean_tx_ring + ice_clean_rx_ring per ring, free DMA buffers.

REQ-16: XDP — ice_xdp / ice_xdp_setup_prog (`ice_main.c:2983` / `:2899`):
- ice_xdp: only ICE_VSI_PF or ICE_VSI_SF; mutex_lock(&vsi->xdp_state_lock).
  - XDP_SETUP_PROG: ice_xdp_setup_prog(vsi, xdp->prog, xdp->extack).
  - XDP_SETUP_XSK_POOL: ice_xsk_pool_setup(vsi, xdp->xsk.pool, xdp->xsk.queue_id).
- ice_xdp_setup_prog:
  - frame_size = mtu + ICE_ETH_PKT_HDR_PAD; if prog && !xdp_has_frags && frame_size > ice_max_xdp_frame_size(vsi): -EOPNOTSUPP.
  - Hot-swap path: if ice_is_xdp_ena_vsi(vsi) == !!prog || ICE_VSI_REBUILD_PENDING — ice_vsi_assign_bpf_prog(vsi, prog); return 0.
  - if_running = netif_running && !test_and_set_bit(ICE_VSI_DOWN); if if_running: ice_down(vsi).
  - if !xdp_ena && prog: ice_vsi_determine_xdp_res(vsi) → ice_prepare_xdp_rings(vsi, prog, ICE_XDP_CFG_FULL) → xdp_features_set_redirect_target(vsi->netdev, true).
  - else if xdp_ena && !prog: xdp_features_clear_redirect_target → ice_destroy_xdp_rings(vsi, ICE_XDP_CFG_FULL).
  - if if_running: ice_up(vsi); if !ret && prog: ice_vsi_rx_napi_schedule.
- netdev->xdp_features = NETDEV_XDP_ACT_BASIC | NETDEV_XDP_ACT_REDIRECT | NETDEV_XDP_ACT_XSK_ZEROCOPY | NETDEV_XDP_ACT_RX_SG; xdp_zc_max_segs = ICE_MAX_BUF_TXD.

REQ-17: AF_XDP — ice_xsk_pool_setup / ice_xsk_wakeup / ice_xmit_zc / ice_clean_rx_irq_zc (`ice_xsk.c`):
- pool_setup binds an `xsk_buff_pool` to (vsi, qid); on bind: re-prepare Rx+Tx ring of qid; allocate xdp_buff array; map pool DMA; on unbind: revert.
- wakeup: trigger NAPI on qid for the xsk-pool TX path.
- xmit_zc: drain UMEM tx-ring, fill HW Tx descriptors, ring doorbell.
- clean_rx_irq_zc: drain HW Rx, run XDP, post to UMEM rx-ring.

REQ-18: SR-IOV — ice_sriov_configure / ice_alloc_vfs / ice_free_vfs (`ice_sriov.c`):
- ice_sriov_configure(pdev, num_vfs):
  - if num_vfs == 0 && !pci_num_vf(pdev): ice_free_vfs + pci_disable_sriov; return 0.
  - if num_vfs > pci_sriov_get_totalvfs: -EINVAL.
  - Validate `ICE_FLAG_SRIOV_CAPABLE`.
  - ice_alloc_vfs(pf, num_vfs) — alloc `struct ice_vf *vfs[num_vfs]`; for each VF: alloc VSI (ICE_VSI_VF), program default MAC, register virtchnl mailbox handlers.
  - pci_enable_sriov(pdev, num_vfs).
  - Set ICE_FLAG_SRIOV_ENA.
- VF NDO ops (`ice_netdev_ops`): set_vf_mac, get_vf_cfg, set_vf_trust, set_vf_vlan, set_vf_link_state, set_vf_rate, set_vf_spoofchk, get_vf_stats.
- VFLR (Function Level Reset) trapped in OICR (PFINT_OICR_VFLR_M) → ice_service_task processes ICE_VFLR_EVENT_PENDING → per-VF reset via virtchnl.

REQ-19: DCB (Data Center Bridging) — ice_init_pf_dcb / ice_dcbnl_setup (`ice_dcb*.c`):
- ETS bandwidth allocation per-TC.
- PFC (Priority Flow Control) per-priority pause.
- IEEE-LLDP / CEE-LLDP exchange via firmware-LLDP-agent or kernel-LLDP-agent.
- Non-fatal on init failure (clear DCB_CAPABLE / DCB_ENA).
- `ice_cfg_lldp_mib_change` registers for LLDP MIB-change AdminQ events.

REQ-20: ETF (Earliest TxTime First) — ice_offload_txtime (`ice_main.c:9348`):
- TC_SETUP_QDISC_ETF → ice_offload_txtime(netdev, qopt).
- if !ICE_F_TXTIME: -EOPNOTSUPP.
- if qopt->queue < 0 || qopt->queue >= vsi->num_txq: -EINVAL.
- if enable: set_bit(qopt->queue, pf->txtime_txqs); else clear_bit.
- if netif_running: ice_cfg_txtime(vsi->tx_rings[qopt->queue]).
- Per-skb TxTime carried in skb->tstamp (CLOCK_TAI) consumed by HW.

REQ-21: RDMA / IDC (`ice_idc.c`):
- ice_init_rdma(pf): allocate aux-bus device `iidc_rdma_core_dev_info`; register as `auxiliary_device` named "intel_rdma_x:y"; on bus-probe `irdma` driver attaches.
- ice_rdma_finalize_setup(pf): publish per-VSI/qos info; plug aux dev.
- ice_deinit_rdma(pf): unplug aux dev; teardown IDC ops.
- ICE_FLAG_RDMA_ENA controlled by capability + module param + devlink.
- Per-RDMA MSI-X carved from PF total (`pf->num_rdma_msix`).

REQ-22: FDIR (Flow Director) — ice_init_fdir + ice_ethtool_fdir.c:
- AdminQ-configured perfect-match filters for L3 / L4 5-tuple, GTP-U, PFCP, ECPRI, etc.
- ethtool: `ETHTOOL_SRXCLSRLINS` / `ETHTOOL_SRXCLSRLDEL` → ice_fdir_add_fltr / ice_fdir_del_fltr.
- aRFS (`ice_arfs.c`): kernel-RFS auto-inserts FDIR filters for established flows.
- Hash-based RSS (`ice_set_rss_lut` / `ice_set_rss_key` / `ice_set_rss_hfunc` (Toeplitz / Simple XOR / Symmetric Toeplitz)) is orthogonal — RSS routes to queue, FDIR overrides for specific 5-tuples.

REQ-23: DDP (Dynamic Device Personalization) — ice_request_fw / ice_load_pkg / ice_log_pkg_init:
- Firmware request: `request_firmware(&firmware, "intel/ice/ddp/ice.pkg", dev)`.
- `MODULE_FIRMWARE(ICE_DDP_PKG_FILE)` declares dependency at module-build time.
- DDP package is parsed and segments are written via AdminQ to flex-pipeline parser + switch + RSS + FDIR engines; ICE_DDP_PKG_SUCCESS / SAME_VERSION_ALREADY_LOADED / ALREADY_LOADED_NOT_SUPPORTED / COMPATIBLE_ALREADY_LOADED / FW_MISMATCH / INVALID_FILE / FILE_VERSION_TOO_HIGH / FILE_VERSION_TOO_LOW / FILE_SIGNATURE_INVALID / FILE_REVISION_TOO_LOW / LOAD_ERROR.
- Failure-not-fatal → driver continues in "safe mode" (XDP refused; only `ice_netdev_safe_mode_ops`).
- DDP is what supplies the protocol-parser definitions consumed by RSS/FDIR/flex-pipeline; without it the device only supports a tiny baseline.

REQ-24: Service task (`ice_service_task` + `ice_service_task_schedule`):
- Workqueue-deferred handler for events that cannot run in hard-IRQ:
  - ICE_ADMINQ_EVENT_PENDING — drain AdminQ events (`ice_clean_adminq_subtask`).
  - ICE_MAILBOXQ_EVENT_PENDING — drain VF virtchnl.
  - ICE_SIDEBANDQ_EVENT_PENDING — drain sideband (PHY / Sensor).
  - ICE_MDD_EVENT_PENDING — Malicious-Driver-Detection handling (VF reset on offender).
  - ICE_VFLR_EVENT_PENDING — process per-VF FLR.
  - ICE_RESET_OICR_RECV / ICE_PFR_REQ / ICE_CORER_REQ / ICE_GLOBR_REQ — orchestrate `ice_do_reset` + `ice_rebuild`.
  - ICE_AUX_ERR_PENDING — RDMA aux-dev error report.
  - Link-state propagation; periodic `serv_tmr` re-arm.

REQ-25: PCIe AER / EEH (`ice_pci_err_handler`):
- ice_pci_err_detected: set ICE_PFR_REQ; ice_prepare_for_reset.
- ice_pci_err_reset_prepare: ice_remove (without unbind).
- ice_pci_err_reset_done: hw reset + ice_probe-equivalent rebuild.
- ice_pci_err_resume: clear error state; ice_service_task_restart.

REQ-26: PM (`ice_pm_ops` = `SIMPLE_DEV_PM_OPS(ice_suspend, ice_resume)`):
- ice_suspend: if !pf_state_is_nominal: -EBUSY; ice_service_task_stop; ice_unplug_aux_dev + ice_deinit_rdma; set ICE_SUSPENDED; ice_prepare_for_shutdown; ice_setup_mc_magic_wake.
- ice_resume: re-init interrupt scheme via `ice_reinit_interrupt_scheme` (which itself calls `ice_init_interrupt_scheme` + per-VSI `ice_vsi_alloc_q_vectors` + `ice_vsi_map_rings_to_vectors` + `ice_vsi_set_napi_queues` + `ice_req_irq_msix_misc`).

REQ-27: Reset orchestration:
- Hierarchy: PFR (PF-Reset, isolated to this PF) < CORER (Core Reset, all PFs same package) < GLOBR (Global Reset, full reset) < EMPR (Embedded-Management-Processor Reset).
- Software-triggered: writes to GLGEN_RTRIG (PFR) / SW reset registers.
- Hardware-triggered: OICR PFINT_OICR_GRST_M with GLGEN_RSTAT_RESET_TYPE field decode.
- ice_do_reset: ice_prepare_for_reset → trigger → poll GLGEN_RSTAT clear → ice_rebuild (re-init AdminQ, reload DDP, re-create VSIs, re-program scheduler, re-request IRQs, re-arm OICR, ice_service_task_restart).

REQ-28: Switchdev / eSwitch (`ice_eswitch.c`):
- DEVLINK_ESWITCH_MODE_LEGACY (default, VEB/VEPA) ↔ DEVLINK_ESWITCH_MODE_SWITCHDEV (per-VF representor netdevs).
- Representor netdev (`ice_repr`) bridges PF kernel-stack ↔ VF traffic for offload via tc-flower.
- Stored in `pf->eswitch.reprs` (xarray).

REQ-29: LAG (`ice_lag.c`):
- Listens on netdev-notifier for `NETDEV_CHANGEUPPER` over `ice_lag_wq`.
- Configures HW LAG (active-backup / 802.3ad) by reprogramming default switch rules.

REQ-30: PTP / SyncE / GNSS:
- `ice_ptp.c`: registers PHC, exposes ptp_clock_*, drives `PFINT_OICR_TSYN_TX_M` + `PFINT_OICR_TSYN_EVNT_M`; ts_dev_info.ts_ll_int_read uses dedicated low-latency-TS IRQ (ice_ll_ts_intr).
- `ice_dpll.c`: DPLL subsystem hook for SyncE + PTP-clock-source selection.
- `ice_gnss.c`: GNSS-UART tty for receivers gating PHC.

REQ-31: Channel / ADQ (Application Device Queues):
- vsi->tc_map_vsi[ICE_CHNL_MAX_TC]; per-TC sub-VSI with per-application MQPRIO queue groups + dedicated rate / interrupt.
- ndo_setup_tc(TC_SETUP_QDISC_MQPRIO) → ice_setup_tc_mqprio_qdisc.

REQ-32: net_device_ops (`ice_main.c:9777`):
- .ndo_open / .ndo_stop / .ndo_start_xmit / .ndo_select_queue / .ndo_features_check / .ndo_fix_features / .ndo_set_rx_mode / .ndo_set_mac_address / .ndo_validate_addr / .ndo_change_mtu / .ndo_get_stats64 / .ndo_set_tx_maxrate / .ndo_set_vf_spoofchk / .ndo_set_vf_mac / .ndo_get_vf_config / .ndo_set_vf_trust / .ndo_set_vf_vlan / .ndo_set_vf_link_state / .ndo_get_vf_stats / .ndo_set_vf_rate / .ndo_vlan_rx_add_vid / .ndo_vlan_rx_kill_vid / .ndo_setup_tc / .ndo_set_features / .ndo_bridge_getlink / .ndo_bridge_setlink / .ndo_fdb_add / .ndo_fdb_del / .ndo_rx_flow_steer (RFS_ACCEL) / .ndo_tx_timeout / .ndo_bpf / .ndo_xdp_xmit / .ndo_xsk_wakeup / .ndo_hwtstamp_get / .ndo_hwtstamp_set.

REQ-33: Safe-mode ops (`ice_main.c:9765`):
- If DDP failed: .ndo_bpf = ice_xdp_safe_mode (always returns -EOPNOTSUPP with NL_SET_ERR_MSG_MOD).
- Minimal subset: open/stop/start_xmit/set_mac_address/validate_addr/change_mtu/get_stats64/tx_timeout/bpf.

## Acceptance Criteria

- [ ] AC-1: pci_register_driver(&ice_driver) succeeds; modaliases (`ice_pci_tbl`) match E810 / E822 / E823 / E825 / E830 device-id table.
- [ ] AC-2: ice_probe on a valid E810 board: allocates pf, maps BAR0, initializes hw, loads DDP, allocates VSIs, registers netdev `enp*s*f*`, returns 0; failure unrolls in reverse order without leaks.
- [ ] AC-3: ice_remove(pdev): waits for in-progress reset; releases VFs if SR-IOV enabled; tears down devlink, RDMA, features, netdev, VSI, HW; idempotent.
- [ ] AC-4: PF-Reset (PFR) trapped in OICR completes ice_rebuild and resumes traffic on all VSIs.
- [ ] AC-5: ice_napi_poll respects budget; if cleaned >= budget per Rx ring or any Tx ring incomplete, returns budget; else napi_complete_done + ice_net_dim + ice_enable_interrupt.
- [ ] AC-6: ice_misc_intr re-arms via `ice_irq_dynamic_ena` only when returning IRQ_HANDLED; PTP-TS path returns from threaded handler.
- [ ] AC-7: ndo_open → ice_open_internal: configures PHY, calls ice_vsi_open, requests per-vector IRQs (ice_msix_clean_rings), enables NAPI, netif_carrier_on on link-up.
- [ ] AC-8: XDP_SETUP_PROG hot-swap (existing prog vs new prog) replaces program without toggling link; cold-attach allocates XDP-Tx rings = one per NAPI vector.
- [ ] AC-9: XDP_SETUP_XSK_POOL binds an `xsk_buff_pool` to (vsi, queue_id); ice_clean_rx_irq_zc drains; ice_xmit_zc transmits zerocopy; ice_xsk_wakeup triggers NAPI on demand.
- [ ] AC-10: ice_sriov_configure(pdev, N): N VFs appear; each VF has a VSI; ice_set_vf_mac persists across VF reset; pci_disable_sriov on 0.
- [ ] AC-11: VFLR (function-level-reset on a VF): OICR PFINT_OICR_VFLR_M → service-task resets that VF only; siblings unaffected.
- [ ] AC-12: tc qdisc add etf clockid CLOCK_TAI on Tx queue N: ice_offload_txtime sets pf->txtime_txqs bit + ice_cfg_txtime; HW transmits skb at skb->tstamp.
- [ ] AC-13: tc qdisc add mqprio: ice_setup_tc_mqprio_qdisc programs per-TC sub-VSIs; rejected if eswitch-mode==switchdev or RDMA is active.
- [ ] AC-14: ethtool -K eth0 rx-vlan-filter on / vlan add: ice_vlan_rx_add_vid programs HW VLAN filter; aRFS / FDIR ethtool rules installed.
- [ ] AC-15: DCB IEEE getets / setets via dcbnl: per-TC bandwidth + PFC honored; lldptool exchange functional when ICE_FLAG_FW_LLDP_AGENT clear.
- [ ] AC-16: DDP load failure: `ice_log_pkg_init` prints reason; driver enters safe-mode; `ice_netdev_safe_mode_ops` installed; ndo_bpf rejects with the documented error.
- [ ] AC-17: RDMA aux-bus: `auxiliary_bus_devices/intel_rdma_x:y` appears; `irdma` binds; ibverbs `ib_dev` exposed; ice_unload unplugs aux-dev cleanly.
- [ ] AC-18: PTP: phc_ctl /dev/ptpN read returns FW time; ptp4l + Tx-timestamp via PFINT_OICR_TSYN_TX_M / ll_ts irq path; ts_dev_info.ts_ll_int_read = true uses dedicated LL-TS vector.

## Architecture

```
struct Pf {
    pdev: *PciDev,
    adapter: *Adapter,                    // sibling-PF coordinator
    hw: Hw,                               // AdminQ + caps + register access
    vsi: Vec<Option<Box<Vsi>>>,           // pf->vsi[]
    vsi_stats: Vec<Option<Box<VsiStats>>>,
    first_sw: Box<Sw>,                    // FW-default switch
    state: BitMap<IceState>,              // ICE_DOWN / ICE_SERVICE_DIS / ICE_RESET_OICR_RECV / ...
    flags: BitMap<IcePfFlags>,            // ICE_FLAG_RDMA_ENA / ICE_FLAG_SRIOV_ENA / ...
    msix: PfMsix { cur, min, max, total, rest },
    oicr_irq: MsiMap,
    ll_ts_irq: MsiMap,
    avail_txqs: BitMap,
    avail_rxqs: BitMap,
    num_alloc_vsi: u16,
    num_lan_msix: u16,
    num_rdma_msix: u16,
    serv_tmr: Timer,
    serv_tmr_period: Jiffies,
    service_task: WorkStruct,
    int_name: [u8; ICE_INT_NAME_STR_LEN],
    int_name_ll_ts: [u8; ICE_INT_NAME_STR_LEN],
    corer_count: u32,
    globr_count: u32,
    empr_count: u32,
    pfr_count: u32,
    aux_idx: i32,
    cdev_info: *IidcRdmaCoreDevInfo,
    txtime_txqs: BitMap,
    eswitch: Eswitch,
    devlink_port: DevlinkPort,
    wakeup_reason: u32,
    wol_ena: bool,
    msg_enable: u32,
    sw_int_count: u32,
    misc_thread: BitMap<IceMiscThread>,
    oicr_err_reg: u32,
}

struct Vsi {
    netdev: *NetDevice,
    vsw: *Sw,
    back: *Pf,
    rx_rings: Vec<*RxRing>,
    tx_rings: Vec<*TxRing>,
    xdp_rings: Vec<*TxRing>,
    q_vectors: Vec<*QVector>,
    irq_handler: fn(i32, *mut c_void) -> IrqReturn, // ice_msix_clean_rings
    state: BitMap<IceVsiState>,
    num_q_vectors: u16,
    num_txq: u16,
    num_rxq: u16,
    num_xdp_txq: u16,
    alloc_txq: u16,
    alloc_rxq: u16,
    vsi_num: u16,                          // HW absolute
    idx: u16,                              // pf->vsi[idx]
    rss_table_size: u16,
    rss_size: u16,
    rss_hfunc: u8,
    rss_hkey_user: *u8,
    rss_lut_user: *u8,
    info: AqcVsiProps,
    vlan_info: VsiVlanInfo,
    inner_vlan_ops: VsiVlanOps,
    outer_vlan_ops: VsiVlanOps,
    arfs_fltr_list: *HListHead,
    arfs_fltr_cntrs: *ArfsActiveFltrCntrs,
    arfs_lock: SpinLock,
    tc_cfg: TcCfg,
    xdp_prog: *BpfProg,
    xdp_state_lock: Mutex,
    mqprio_qopt: TcMqprioQoptOffload,
    tc_map_vsi: [*Vsi; ICE_CHNL_MAX_TC],
    params: VsiCfgParams { port_info, ch, vf|sf, flags, type },
}

struct QVector {
    vsi: *Vsi,
    v_idx: u16,
    reg_idx: u16,
    num_ring_rx: u8,
    num_ring_tx: u8,
    wb_on_itr: bool,
    intrl: u8,
    napi: NapiStruct,
    rx: RingContainer,
    tx: RingContainer,
    ch: *Channel,
    name: [u8; ICE_INT_NAME_STR_LEN],
    total_events: u16,
    irq: MsiMap,
}
```

`Pf::probe(pdev, ent) -> Result<(), Error>`:
1. /* Reject VF-attempt-on-PF-driver */
2. if pdev.is_virtfn(): return Err(EINVAL).
3. /* kdump-FLR */
4. if is_kdump_kernel(): pci_save_state; pci_clear_master; pcie_flr?; pci_restore_state.
5. pcim_enable_device(pdev)?.
6. pcim_iomap_regions(pdev, 1 << ICE_BAR0, drv_string)?.
7. pf = ice_allocate_pf(dev)?; pf.aux_idx = -1.
8. dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64))?.
9. pci_set_master(pdev); pf.pdev = pdev; pci_set_drvdata(pdev, pf).
10. set_bit(ICE_DOWN, pf.state); set_bit(ICE_SERVICE_DIS, pf.state).
11. /* HW register window */
12. hw = &pf.hw; hw.hw_addr = pcim_iomap_table(pdev)[ICE_BAR0]; pci_save_state.
13. populate hw.vendor_id / device_id / revision_id / subsystem_*; bus.device/func.
14. ice_set_ctrlq_len(hw).
15. pf.msg_enable = netif_msg_init(debug, ICE_DFLT_NETIF_M).
16. /* Recovery-only path */
17. if Hw::is_recovery_mode(hw): return Pf::probe_recovery_mode(pf).
18. /* HW + FW + DDP */
19. Hw::init(hw)?.            // AdminQ + caps + DDP load + sched tree
20. Pf::init_dev_hw(pf).
21. adapter = Adapter::get(pdev)?; pf.adapter = adapter.
22. /* Pre-VSI device init (irq scheme, FDIR pools, MAC) */
23. Pf::init_dev(pf)?.
24. /* VSI array + main VSI + link */
25. Pf::init(pf)?.
26. /* Netdev + features + RDMA — under devl_lock */
27. devl_lock(priv_to_devlink(pf)).
28. Pf::load(pf)?.
29. Pf::init_devlink(pf)?.
30. devl_unlock.
31. return Ok.

`Pf::init(pf) -> Result`:
1. Pf::init_pf(pf)?.                       // bitmaps, workqueue, locks, serv_tmr
2. if hw.mac_type == ICE_MAC_E830: pci_enable_ptm(pdev) (best-effort).
3. Pf::alloc_vsis(pf)?.
4. Pf::init_pf_sw(pf)?.                    // first_sw + ice_pf_vsi_setup
5. Pf::init_wakeup(pf).                    // PFPM_WUS / WoL
6. Pf::init_link(pf)?.                     // PHY + link events
7. Pf::send_version(pf)?.
8. Pf::verify_cacheline_size(pf).
9. if is_safe_mode: Pf::set_safe_mode_vlan_cfg else pcie_print_link_status.
10. clear_bit(ICE_DOWN, ICE_SERVICE_DIS).
11. mod_timer(&pf.serv_tmr, round_jiffies(jiffies + serv_tmr_period)).

`Pf::load(pf) -> Result`:
1. devl_assert_locked.
2. vsi = ice_get_main_vsi(pf).
3. INIT_LIST_HEAD(&vsi.ch_list).
4. Vsi::cfg_netdev(vsi)?.                  // alloc_netdev_mqs + feature flags
5. Dcb::dcbnl_setup(vsi).
6. Pf::init_mac_fltr(pf)?.
7. Pf::devlink_create_pf_port(pf)?.
8. SET_NETDEV_DEVLINK_PORT(vsi.netdev, &pf.devlink_port).
9. Vsi::register_netdev(vsi)?.
10. Tc::indir_block_register(vsi)?.
11. QVector::napi_add_all(vsi).
12. Pf::init_features(pf).                 // PTP/GNSS/DPLL/FDIR/DCB/LAG/HWMON
13. Rdma::init(pf)?.
14. Rdma::finalize_setup(pf).
15. Pf::service_task_restart(pf).
16. clear_bit(ICE_DOWN, pf.state).

`Pf::remove(pdev)`:
1. pf = pci_get_drvdata(pdev).
2. for i in 0..ICE_MAX_RESET_WAIT: if !is_reset_in_progress(pf.state) break; else msleep(100).
3. if Hw::is_recovery_mode(&pf.hw):
   - Pf::service_task_stop(pf).
   - scoped_guard(devl, priv_to_devlink(pf)) { Pf::deinit_devlink(pf) }.
   - return.
4. if test_bit(ICE_FLAG_SRIOV_ENA, pf.flags): set_bit(ICE_VF_RESETS_DISABLED); Sriov::free_vfs(pf).
5. if !is_safe_mode: Arfs::remove(pf).
6. devl_lock.
7. Pf::dealloc_all_dynamic_ports(pf).
8. Pf::deinit_devlink(pf).
9. Pf::unload(pf).
10. devl_unlock.
11. Pf::deinit(pf).
12. Pf::vsi_release_all(pf).
13. Pf::setup_mc_magic_wake(pf); Pf::set_wake(pf).
14. Adapter::put(pdev).
15. Hw::deinit(&pf.hw).
16. Pf::deinit_dev(pf).
17. Pf::aq_cancel_waiting_tasks(pf).
18. set_bit(ICE_DOWN, pf.state).

`Pf::misc_intr(irq, pf) -> IrqReturn`:
1. ret = IRQ_HANDLED.
2. set_bit(ICE_ADMINQ_EVENT_PENDING|ICE_MAILBOXQ_EVENT_PENDING|ICE_SIDEBANDQ_EVENT_PENDING).
3. oicr = rd32(hw, PFINT_OICR); ena_mask = rd32(hw, PFINT_OICR_ENA).
4. if oicr & PFINT_OICR_SWINT_M: pf.sw_int_count++; ena_mask &= ~PFINT_OICR_SWINT_M.
5. if oicr & PFINT_OICR_MAL_DETECT_M: ena_mask &= ~PFINT_OICR_MAL_DETECT_M; set_bit(ICE_MDD_EVENT_PENDING).
6. if oicr & PFINT_OICR_VFLR_M:
   - if ICE_VF_RESETS_DISABLED: clear PFINT_OICR_VFLR_M in PFINT_OICR_ENA.
   - else: ena_mask &= ~PFINT_OICR_VFLR_M; set_bit(ICE_VFLR_EVENT_PENDING).
7. if oicr & PFINT_OICR_GRST_M:
   - reset_type = FIELD_GET(GLGEN_RSTAT_RESET_TYPE_M, rd32(hw, GLGEN_RSTAT)).
   - bump corer_count|globr_count|empr_count.
   - if !test_and_set_bit(ICE_RESET_OICR_RECV, pf.state):
     - set per-type ICE_CORER_RECV|ICE_GLOBR_RECV|ICE_EMPR_RECV.
     - hw.reset_ongoing = true.
8. if oicr & PFINT_OICR_TSYN_TX_M: ret = Ptp::ts_irq(pf).
9. if oicr & PFINT_OICR_TSYN_EVNT_M: read GLTSYN_STAT; if src_tmr_owned: ext_ts_irq |= ...; Ptp::extts_event(pf).
10. if oicr & ICE_AUX_CRIT_ERR (PE_CRITERR | HMC_ERR | PE_PUSH): pf.oicr_err_reg |= oicr; set ICE_AUX_ERR_PENDING.
11. residual = oicr & ena_mask; if residual & (PCI_EXCEPTION | ECC_ERR): set ICE_PFR_REQ.
12. Pf::service_task_schedule(pf).
13. if ret == IRQ_HANDLED: ice_irq_dynamic_ena(hw, None, None).
14. return ret.

`Pf::misc_intr_thread_fn(irq, pf) -> IrqReturn`:
1. if Hw::reset_in_progress(pf.state): goto skip_irq.
2. if test_and_clear_bit(ICE_MISC_THREAD_TX_TSTAMP, pf.misc_thread): Ptp::process_ts(pf).
3. skip_irq: ice_irq_dynamic_ena + ice_flush.
4. if Ptp::tx_tstamps_pending: wr32(PFINT_OICR, PFINT_OICR_TSYN_TX_M); ice_flush.
5. return IRQ_HANDLED.

`QVector::msix_irq(irq, q_vector) -> IrqReturn`:
1. if !q_vector.tx.tx_ring && !q_vector.rx.rx_ring: return IRQ_HANDLED.
2. q_vector.total_events++.
3. napi_schedule(&q_vector.napi).
4. return IRQ_HANDLED.

`QVector::napi_poll(napi, budget) -> i32`:
1. q_vector = container_of(napi, ice_q_vector, napi).
2. clean_complete = true.
3. for tx_ring in q_vector.tx:
   - xsk_pool = READ_ONCE(tx_ring.xsk_pool).
   - wd = if xsk_pool: Xsk::xmit_zc(tx_ring, xsk_pool); else if ring_is_xdp(tx_ring): true; else TxRing::clean(tx_ring, budget).
   - if !wd: clean_complete = false.
4. if budget ≤ 0: return budget.
5. budget_per_ring = if num_ring_rx > 1 { max(budget / num_ring_rx, 1) } else budget.
6. work_done = 0.
7. for rx_ring in q_vector.rx:
   - xsk_pool = READ_ONCE(rx_ring.xsk_pool).
   - cleaned = if xsk_pool { Xsk::clean_rx_zc(rx_ring, xsk_pool, budget_per_ring) } else RxRing::clean(rx_ring, budget_per_ring).
   - work_done += cleaned.
   - if cleaned ≥ budget_per_ring: clean_complete = false.
8. if !clean_complete: QVector::set_wb_on_itr(q_vector); return budget.
9. if napi_complete_done(napi, work_done): QVector::net_dim(q_vector); QVector::enable_interrupt(q_vector); else QVector::set_wb_on_itr(q_vector).
10. return min(work_done, budget - 1).

`Vsi::xdp_setup_prog(vsi, prog, extack) -> Result`:
1. frame_size = vsi.netdev.mtu + ICE_ETH_PKT_HDR_PAD.
2. if prog && !prog.aux.xdp_has_frags && frame_size > ice_max_xdp_frame_size(vsi): NL_SET_ERR_MSG_MOD; return -EOPNOTSUPP.
3. /* Hot-swap shortcut */
4. if is_xdp_ena_vsi(vsi) == !!prog || test_bit(ICE_VSI_REBUILD_PENDING, vsi.state):
   - Vsi::assign_bpf_prog(vsi, prog).
   - return Ok.
5. if_running = netif_running(vsi.netdev) && !test_and_set_bit(ICE_VSI_DOWN, vsi.state).
6. if if_running: Vsi::down(vsi)?.
7. if !is_xdp_ena_vsi(vsi) && prog:
   - Vsi::determine_xdp_res(vsi)?.
   - Vsi::prepare_xdp_rings(vsi, prog, ICE_XDP_CFG_FULL)?.
   - xdp_features_set_redirect_target(vsi.netdev, true).
8. else if is_xdp_ena_vsi(vsi) && !prog:
   - xdp_features_clear_redirect_target(vsi.netdev).
   - Vsi::destroy_xdp_rings(vsi, ICE_XDP_CFG_FULL)?.
9. if if_running: Vsi::up(vsi).
10. if !ret && prog: Vsi::rx_napi_schedule(vsi).

`Sriov::configure(pdev, num_vfs) -> i32`:
1. pf = pci_get_drvdata(pdev).
2. if num_vfs == 0:
   - if pci_num_vf(pdev) > 0: Sriov::free_vfs(pf); pci_disable_sriov(pdev).
   - clear_bit(ICE_FLAG_SRIOV_ENA, pf.flags).
   - return 0.
3. if num_vfs > pci_sriov_get_totalvfs(pdev): return -EINVAL.
4. if !test_bit(ICE_FLAG_SRIOV_CAPABLE, pf.flags): return -EOPNOTSUPP.
5. Sriov::alloc_vfs(pf, num_vfs)?.
6. pci_enable_sriov(pdev, num_vfs)?.
7. set_bit(ICE_FLAG_SRIOV_ENA, pf.flags).
8. return num_vfs.

`Tx::offload_txtime(netdev, qopt_off) -> i32` (ETF):
1. pf = np.vsi.back.
2. if !is_feature_supported(pf, ICE_F_TXTIME): return -EOPNOTSUPP.
3. qopt = qopt_off; if qopt == NULL || qopt.queue < 0 || qopt.queue ≥ vsi.num_txq: return -EINVAL.
4. if qopt.enable: set_bit(qopt.queue, pf.txtime_txqs); else clear_bit.
5. if netif_running(vsi.netdev):
   - tx_ring = vsi.tx_rings[qopt.queue].
   - TxRing::cfg_txtime(tx_ring)?.
6. netdev_info(en/dis TxTime queue N).
7. return 0; on err undo pf.txtime_txqs.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `probe_unrolls_on_failure` | INVARIANT | per-Pf::probe: any sub-step failure unrolls in reverse (deinit_dev / adapter_put / deinit_hw / etc.) — no orphan pf, no leaked BAR. |
| `remove_idempotent_under_reset_wait` | INVARIANT | per-Pf::remove: waits up to ICE_MAX_RESET_WAIT × 100ms for reset; never frees while reset_in_progress. |
| `irq_alloc_paired_with_free` | INVARIANT | per-(`ice_alloc_irq`,`ice_free_irq`): exact pair count across probe/remove + suspend/resume. |
| `napi_poll_respects_budget` | INVARIANT | per-QVector::napi_poll: ret ≤ budget; if clean_complete: ret < budget. |
| `napi_complete_paired_with_enable` | INVARIANT | per-NAPI: napi_complete_done implies subsequent ice_enable_interrupt — never both ice_set_wb_on_itr and enable_interrupt. |
| `msix_clean_rings_only_schedules_napi` | INVARIANT | per-QVector::msix_irq: handler does no register write; only napi_schedule. |
| `oicr_reset_bit_handshake` | INVARIANT | per-Pf::misc_intr: ICE_RESET_OICR_RECV set under test_and_set_bit; per-type-bit set only once per reset. |
| `xdp_hot_swap_no_link_toggle` | INVARIANT | per-Vsi::xdp_setup_prog: if is_xdp_ena == !!prog or REBUILD_PENDING — assign without down/up. |
| `xdp_setup_holds_xdp_state_lock` | INVARIANT | per-Vsi::xdp: mutex_lock(xdp_state_lock) for entire XDP_SETUP_*. |
| `sriov_no_enable_without_capable` | INVARIANT | per-Sriov::configure: ICE_FLAG_SRIOV_CAPABLE required pre-pci_enable_sriov. |
| `vsi_state_under_lock` | INVARIANT | per-Vsi::up/down/open/close: test_and_set/clear_bit on vsi.state under proper ordering. |
| `safe_mode_only_safe_ops` | INVARIANT | per-DDP-failure: netdev.netdev_ops == ice_netdev_safe_mode_ops; ndo_bpf returns -EOPNOTSUPP. |
| `txtime_within_num_txq` | INVARIANT | per-Tx::offload_txtime: qopt.queue ∈ [0, vsi.num_txq). |
| `ll_ts_only_if_capable` | INVARIANT | per-Pf::req_irq_msix_misc: LL-TS IRQ requested only if ts_dev_info.ts_ll_int_read. |

### Layer 2: TLA+

`drivers/net/ethernet/intel/ice/ice.tla`:
- Per-probe → init_hw (DDP load) → init → load → ready → unload → deinit → deinit_hw → done.
- Per-OICR event arrival; per-NAPI ring-clean; per-reset (PFR / CORER / GLOBR / EMPR) → rebuild → ready.
- Per-SR-IOV enable/disable + VFLR-per-VF; per-XDP-setup hot-swap and cold-attach.
- Properties:
  - `safety_no_traffic_before_ready` — per-ice_probe: ndo_open returns -EBUSY before clear ICE_DOWN.
  - `safety_reset_serialized` — per-OICR: at most one PFR/CORER/GLOBR/EMPR in-flight per pf.
  - `safety_xdp_state_lock_excludes_parallel_setup` — per-XDP: xdp_state_lock serializes XDP_SETUP_PROG with XDP_SETUP_XSK_POOL.
  - `safety_vf_reset_local_only` — per-VFLR(vf_i): does not perturb sibling VFs nor PF VSI.
  - `safety_ddp_failure_implies_safe_mode` — per-load_pkg ≠ SUCCESS / SAME_VERSION / COMPATIBLE: safe-mode ops installed.
  - `liveness_reset_completes` — per-reset_in_progress: eventually ice_rebuild completes and clear_bit(ICE_DOWN).
  - `liveness_napi_completes` — per-NAPI rescheduled bounded by net-dim; eventually clean_complete or budget-exhaust.
  - `liveness_admin_q_drained` — per-OICR ADMINQ_PENDING: service_task drains AdminQ events.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pf::probe` post: ok ⟹ pf registered + main VSI netdev IFF_UP-eligible | `Pf::probe` |
| `Pf::probe` err: pf freed + BAR unmapped + no IRQ leaked | `Pf::probe` |
| `Pf::remove` post: SR-IOV disabled + RDMA aux-dev unplugged + all VSIs released | `Pf::remove` |
| `Pf::misc_intr` post: at most one of {PFR, CORER, GLOBR, EMPR} reset-type-bit set per OICR | `Pf::misc_intr` |
| `Pf::req_irq_msix_misc` post: oicr_irq.virq IRQ-registered + OICR enabled in HW | `Pf::req_irq_msix_misc` |
| `QVector::napi_poll` post: work_done ≤ budget; clean_complete ⟹ ret < budget | `QVector::napi_poll` |
| `Vsi::xdp_setup_prog` post: ok ⟹ vsi.xdp_prog == prog (or NULL) + XDP rings consistent | `Vsi::xdp_setup_prog` |
| `Sriov::configure(0)` post: pci_num_vf(pdev) == 0 | `Sriov::configure` |
| `Tx::offload_txtime` post: ok ⟹ bit(qopt.queue) in pf.txtime_txqs == qopt.enable | `Tx::offload_txtime` |
| `Hw::init` post: DDP-state ∈ {SUCCESS, SAME_VERSION_ALREADY_LOADED, COMPATIBLE_ALREADY_LOADED} ∨ pf in safe-mode | `Hw::init` |

### Layer 4: Verus/Creusot functional

`Per-PCIe-probe ⟹ init_hw (AdminQ + DDP + sched-tree) ⟹ alloc_vsis ⟹ init_pf_sw (default-switch + main VSI) ⟹ init_link (PHY) ⟹ cfg_netdev ⟹ register_netdev ⟹ napi_add ⟹ init_features (PTP/GNSS/DPLL/FDIR/DCB/LAG) ⟹ init_rdma (aux-bus plug) ⟹ ready` semantic equivalence with the Linux ice driver — per `Documentation/networking/device_drivers/ethernet/intel/ice.rst` and the E810 datasheet (FPK / CPK / IPK family). Per-OICR + per-NAPI + per-AdminQ event-loop equivalence; per-SR-IOV + per-VFLR + per-XDP-DRV + per-AF_XDP-ZC + per-DCB + per-PTP + per-RDMA + per-ETF + per-MQPRIO + per-eswitch behavioral equivalence.

## Hardening

(Inherits row-1 features from `drivers/net/00-overview.md` § Hardening.)

ice driver reinforcement:

- **Per-PCIe-BAR0 iomap via pcim_*** — defense against per-leaked-MMIO-mapping on probe failure.
- **Per-DMA mask DMA_BIT_MASK(64) enforced** — defense against per-truncated-DMA addresses on >4 GiB systems.
- **Per-pdev->is_virtfn rejected at PF entry** — defense against per-VF-driver mismatch.
- **Per-kdump pcie_flr-before-enable** — defense against per-pending-DMA-clears-after-host-crash machine-check.
- **Per-OICR re-arm only when IRQ_HANDLED** — defense against per-storm under critical-error.
- **Per-PFINT_OICR_ENA scrubbed on VFLR_RESETS_DISABLED** — defense against per-VFLR-flood during shutdown.
- **Per-OICR PCI_EXCEPTION / ECC_ERR → ICE_PFR_REQ (not silent)** — defense against per-silent-data-corruption.
- **Per-reset-in-progress wait in remove (ICE_MAX_RESET_WAIT × 100 ms)** — defense against per-tear-down-during-reset.
- **Per-AdminQ + Mailbox + Sideband + OICR enables grouped via ice_ena_ctrlq_interrupts** — defense against per-partial-enable.
- **Per-DDP load failure ⟹ safe-mode netdev_ops** — defense against per-XDP-with-unparsed-protocols.
- **Per-DDP package signature validation in FW (ICE_DDP_PKG_FILE_SIGNATURE_INVALID)** — defense against per-tampered firmware.
- **Per-XDP state under vsi->xdp_state_lock** — defense against per-XDP-PROG vs XDP-XSK-POOL race.
- **Per-VF spoofchk default-on + admin-MAC enforced** — defense against per-VF MAC spoofing.
- **Per-VF MDD (malicious-driver-detection) triggers VF-reset on offender** — defense against per-VF Tx/Rx-malformed-descriptor abuse.
- **Per-RDMA aux-bus unplug before remove** — defense against per-iWARP/RoCE UAF on driver tear-down.
- **Per-eswitch-mode-switchdev gates MQPRIO** — defense against per-conflicting-offload.
- **Per-PTP TSYN_TX threaded handler** — defense against per-irq-context sleep when reading FW Tx timestamp.
- **Per-LAG operations on dedicated ordered workqueue (ice_lag_wq)** — defense against per-LAG-reentrancy.
- **Per-FDIR / aRFS filter-count caps + per-VSI flow-table isolation** — defense against per-noisy-neighbor flow-table exhaustion.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- iavf driver (Intel Adaptive VF, separate Tier-3 if expanded)
- irdma driver (`drivers/infiniband/hw/irdma/`) — RDMA verbs side; covered in `drivers/infiniband/00-overview.md`
- Generic XDP / AF_XDP core (`net/xdp/`) — covered in `net/xdp.md` Tier-3 if expanded
- Page-pool API (`net/core/page_pool.c`) — covered in `net/page-pool.md` Tier-3 if expanded
- Generic NAPI (`net/core/dev.c`) — covered in `net/dev-core.md` Tier-3 if expanded
- TC (traffic control) framework, qdisc/classifier (`net/sched/`) — covered separately
- devlink subsystem (`net/devlink/`) — covered separately
- PTP / PHC core (`drivers/ptp/`) — covered separately
- DCB netlink core (`net/dcb/`) — covered separately
- PCIe MSI-X / AER core (`drivers/pci/`) — covered in `drivers/pci/00-overview.md`
- Auxiliary bus core (`drivers/base/auxiliary.c`) — covered in `drivers/base/00-overview.md`
- Implementation code
