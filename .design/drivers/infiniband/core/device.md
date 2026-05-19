# Tier-3: drivers/infiniband/core/device.c — IB device registration

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/infiniband/00-overview.md
upstream-paths:
  - drivers/infiniband/core/device.c (~3155 lines)
  - include/rdma/ib_verbs.h (struct ib_device, struct ib_client, struct ib_device_ops, struct ib_port_attr, struct ib_port_data)
  - include/rdma/restrack.h (rdma_restrack_init)
  - include/uapi/rdma/rdma_user_ioctl_cmds.h (uverbs ABI)
-->

## Summary

The IB core device layer is the registration and lookup spine of the kernel RDMA subsystem. Per-HCA driver (mlx5, hfi1, qib, irdma, bnxt_re, ...) allocates a `struct ib_device` via `_ib_alloc_device` (macro `ib_alloc_device`), populates `ops` via `ib_set_device_ops`, then calls `ib_register_device(device, name, dma_device)`. The core registers the device in a global xarray (`devices`), assigns a stable index, sets up sysfs (`/sys/class/infiniband/<name>/`), creates per-port attribute groups, allocates uverbs char devs (`/dev/infiniband/uverbs<N>`), populates the GID/PKey cache (`ib_cache_setup_one`), and finally invokes every registered `ib_client.add()` so RDMA verb consumers (ipoib, rdma_cm, srp, iser, ucm) can attach. Per-port state: `ib_port_attr` (LID, MTU, state, link layer, GID/PKey table lengths). Per-`struct ib_device_ops` ~150-entry vtable describes the entire kernel-verbs surface; `ib_device_check_mandatory` asserts that at least the 16 kverbs-required ops (query_device, query_port, alloc_pd, dealloc_pd, create_qp, modify_qp, destroy_qp, post_send, post_recv, create_cq, destroy_cq, poll_cq, req_notify_cq, get_dma_mr, reg_user_mr, dereg_mr, get_port_immutable) are present. Critical for: uverbs-ABI stability, hot-add/hot-remove of HCAs, net-namespace isolation, GID cache freshness driving rdma_cm routing.

This Tier-3 covers `drivers/infiniband/core/device.c` (~3155 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ib_device` | per-HCA core object | `IbDevice` |
| `struct ib_device_ops` | per-driver vtable (~150 ops) | `IbDeviceOps` |
| `struct ib_client` | per-consumer add/remove cb | `IbClient` |
| `struct ib_port_data` | per-port runtime state | `IbPortData` |
| `struct ib_port_attr` | per-port attributes (LID, MTU, state) | `IbPortAttr` |
| `_ib_alloc_device()` | per-driver alloc + init | `IbDevice::alloc` |
| `ib_dealloc_device()` | per-driver free | `IbDevice::dealloc` |
| `ib_register_device()` | per-HCA registration | `IbDevice::register` |
| `ib_unregister_device()` | per-HCA unregistration | `IbDevice::unregister` |
| `ib_unregister_device_and_put()` | per-unregister + put | `IbDevice::unregister_and_put` |
| `ib_unregister_device_queued()` | per-async unregister | `IbDevice::unregister_queued` |
| `ib_unregister_driver()` | per-driver-id mass-unreg | `IbDevice::unregister_driver` |
| `ib_set_device_ops()` | per-vtable splice | `IbDevice::set_ops` |
| `ib_register_client()` | per-consumer registration | `IbClient::register` |
| `ib_unregister_client()` | per-consumer unregistration | `IbClient::unregister` |
| `ib_set_client_data()` | per-client priv-pointer | `IbDevice::set_client_data` |
| `ib_get_client_data()` | per-client priv-lookup | `IbDevice::get_client_data` |
| `ib_register_event_handler()` | per-async-event hook | `IbDevice::register_event_handler` |
| `ib_unregister_event_handler()` | per-async-event hook | `IbDevice::unregister_event_handler` |
| `ib_dispatch_event_clients()` | per-async-event fanout | `IbDevice::dispatch_event_clients` |
| `ib_device_check_mandatory()` | per-kverbs-required-ops check | `IbDevice::check_mandatory` |
| `ib_device_get_by_index()` | per-xarray lookup | `IbDevice::get_by_index` |
| `ib_device_get_by_netdev()` | per-netdev lookup | `IbDevice::get_by_netdev` |
| `ib_device_put()` | per-refcount dec | `IbDevice::put` |
| `ib_device_rename()` | per-rename | `IbDevice::rename` |
| `ib_device_set_netdev()` | per-port netdev binding | `IbDevice::set_netdev` |
| `ib_device_get_netdev()` | per-port netdev fetch | `IbDevice::get_netdev` |
| `ib_query_port()` / `__ib_query_port()` / `iw_query_port()` | per-port attribute query | `IbDevice::query_port` |
| `ib_query_pkey()` | per-port PKey lookup | `IbDevice::query_pkey` |
| `ib_find_pkey()` | per-port PKey reverse-find | `IbDevice::find_pkey` |
| `ib_find_gid()` | per-port GID reverse-find | `IbDevice::find_gid` |
| `ib_modify_device()` | per-system-image-guid / node-desc | `IbDevice::modify_device` |
| `ib_modify_port()` | per-port-cap-mask | `IbDevice::modify_port` |
| `ib_port_immutable_read()` | per-port immutable snapshot | `IbDevice::port_immutable_read` |
| `setup_port_data()` / `alloc_port_data()` / `verify_immutable()` | per-port-array init | `IbDevice::setup_ports` |
| `add_client_context()` / `remove_client_context()` | per-client priv-array mgmt | `IbDevice::client_ctx` |
| `assign_name()` / `alloc_name()` | per-name conflict-free assignment | `IbDevice::assign_name` |
| `setup_device()` / `enable_device_and_get()` / `disable_device()` | per-registration phase machine | `IbDevice::lifecycle` |
| `rdma_dev_init_net()` / `rdma_dev_exit_net()` | per-netns pernet ops | `IbDevice::netns_ops` |
| `rdma_compatdev_set()` / `add_one_compat_dev()` / `remove_one_compat_dev()` | per-netns compat-shadow | `IbDevice::compat_dev` |
| `rdma_dev_change_netns()` / `ib_device_set_netns_put()` | per-device netns migrate | `IbDevice::change_netns` |
| `ib_device_set_dim()` | per-DIM (dyn-interrupt-mod) toggle | `IbDevice::set_dim` |
| `rdma_dev_has_raw_cap()` | per-CAP_NET_RAW check | `IbDevice::has_raw_cap` |
| `rdma_dev_access_netns()` | per-netns access policy | `IbDevice::access_netns` |
| `ib_enum_roce_netdev()` / `ib_enum_all_roce_netdevs()` | per-RoCE-netdev iter | `IbDevice::enum_roce` |
| `ib_dispatch_port_state_event()` / `handle_port_event()` / `ib_netdevice_event()` | per-netdev-event bridge | `IbDevice::port_state_event` |
| `ib_add_sub_device()` / `ib_del_sub_device_and_put()` | per-sub-device (smi/gsi-share) | `IbDevice::sub_device` |
| `ib_security_change()` / `ib_policy_change_task()` | per-LSM-policy-change | `IbDevice::security_change` |
| `prevent_dealloc_device()` | per-error-flow ownership flip | `IbDevice::prevent_dealloc` |
| `ib_unregister_work()` | per-async-unregister worker | `IbDevice::unregister_work` |
| `xarray devices` (XA_FLAGS_ALLOC) | per-system device registry | `IbDevice::registry` |
| `xarray clients` (XA_FLAGS_ALLOC) | per-system client registry | `IbClient::registry` |
| `rwsem devices_rwsem` / `rwsem clients_rwsem` | per-registry mutex | `IbDevice::sems` |
| `xarray rdma_nets` | per-netns rdma-net table | `IbDevice::netns_table` |
| `class ib_class` | per-/sys/class/infiniband | shared sysfs |
| `notifier_block ibdev_lsm_nb` | per-LSM policy notifier | `IbDevice::lsm_notifier` |
| `notifier_block nb_netdevice` | per-netdev event notifier | `IbDevice::netdev_notifier` |

## Compatibility contract

REQ-1: struct ib_device (subset):
- dev: embedded struct device (sysfs / driver-model anchor).
- ops: struct ib_device_ops (per-driver vtable).
- name: device name (e.g., "mlx5_0").
- index: u32 stable xarray index, also exposed via /sys/class/infiniband/<name>/index.
- node_type: RDMA_NODE_IB_CA / _SWITCH / _ROUTER / _RNIC / _USNIC / _USNIC_UDP / _UNSPECIFIED.
- node_guid / sys_image_guid: per-IBA u64 identifiers (big-endian on the wire).
- attrs: struct ib_device_attr (max_qp, max_cq, max_mr, max_pd, kernel_cap_flags, ...).
- phys_port_cnt: number of physical ports (1..N; port number is 1-based in IBA).
- num_comp_vectors: number of completion vectors (MSI-X-like).
- port_data: per-port array, indexed 0..phys_port_cnt-1, but exposed as port_num 1..phys_port_cnt.
- coredev: struct ib_core_device (per-netns shadow + sysfs anchor).
- compat_devs: xarray of per-netns shadow devices (when ib_devices_shared_netns=0).
- client_data: xarray (XA_FLAGS_ALLOC) of per-client void* + CLIENT_DATA_REGISTERED mark.
- client_data_rwsem: per-device rwsem guarding client_data.
- event_handler_list: list_head of struct ib_event_handler.
- event_handler_rwsem: rwsem guarding event_handler_list.
- unregistration_lock: mutex serializing unreg paths.
- unreg_completion: completion fired when refcount hits zero.
- unregistration_work: workqueue item for ib_unregister_device_queued.
- refcount: refcount_t (started at 1 by alloc, dropped to 0 to release).
- cache_lock: rwlock_t guarding GID/PKey cache.
- cq_pools / cq_pools_lock: per-poll-ctx CQ pool list-heads.
- subdev_list / subdev_list_head / subdev_lock: per-sub-device chain.
- uverbs_cmd_mask: bitmask of supported IB_USER_VERBS_CMD_*.
- kverbs_provider: bool — true iff all mandatory_table ops set.
- groups[]: device-level attribute groups (ib_dev_attr_group + ops->device_group).
- dma_device: backing dma-capable struct device (PCI, platform, or NULL = virt-only).

REQ-2: struct ib_device_ops (vtable; ~150 entries, the SET_DEVICE_OP list in ib_set_device_ops is canonical):
- driver_id: enum rdma_driver_id (RDMA_DRIVER_MLX5, _HFI1, _IRDMA, _BNXT_RE, _MLX4, _CXGB4, _QIB, _OPA_VNIC, _USNIC, _USNIC_UDP, _RXE, _SIW, _EFA, _MANA, _ERDMA, _UNKNOWN, ...).
- owner: struct module* — module-ref discipline against unload-while-used.
- uverbs_abi_ver: u32 driver-specific uverbs ABI version.
- uverbs_no_driver_id_binding / uverbs_robust_udata: u8 bool quirks.
- Object-size fields (SET_OBJ_SIZE): size_ib_ah, size_ib_counters, size_ib_cq, size_ib_dmah, size_ib_mw, size_ib_pd, size_ib_qp, size_ib_rwq_ind_table, size_ib_srq, size_ib_ucontext, size_ib_xrcd, size_rdma_counter — drive rdma_zalloc_drv_obj allocator.
- Kernel-verb method pointers (subset, see canonical SET_DEVICE_OP list in REQ-15): add_gid, alloc_mr, alloc_pd, alloc_ucontext, alloc_xrcd, attach_mcast, create_ah, create_cq, create_qp, create_srq, create_user_ah, create_user_cq, create_wq, dealloc_pd, dealloc_ucontext, dealloc_xrcd, del_gid, dereg_mr, destroy_ah, destroy_cq, destroy_qp, destroy_srq, destroy_wq, detach_mcast, drain_rq, drain_sq, fill_res_*_entry, get_dev_fw_str, get_dma_mr, get_hw_stats, get_link_layer, get_netdev, get_port_immutable, mmap, modify_ah, modify_cq, modify_device, modify_port, modify_qp, modify_srq, peek_cq, poll_cq, post_recv, post_send, post_srq_recv, process_mad, query_ah, query_device, query_gid, query_pkey, query_port, query_qp, query_srq, query_ucontext, reg_user_mr, reg_user_mr_dmabuf, req_notify_cq, rereg_user_mr, resize_user_cq, report_port_event, dealloc_driver, enable_driver, ...
- Mandatory subset (ib_device_check_mandatory) gates kverbs_provider = true: query_device, query_port, alloc_pd, dealloc_pd, create_qp, modify_qp, destroy_qp, post_send, post_recv, create_cq, destroy_cq, poll_cq, req_notify_cq, get_dma_mr, reg_user_mr, dereg_mr, get_port_immutable.

REQ-3: struct ib_client:
- name: const char* (e.g., "ipoib", "rdma_cm", "ucm", "srp", "iser", "rds_rdma").
- add(ibdev) -> int: per-device add hook (call back to set_client_data).
- remove(ibdev, client_data): per-device remove hook (must not fail).
- rename(dev, client_data): optional rename hook.
- get_nl_info / get_global_nl_info: optional netlink-ID resolution.
- get_net_dev_by_params: optional rdma_cm net_dev resolver (IP-over-IB).
- uses: refcount_t (1 + per-attached-device).
- uses_zero: completion fired when uses → 0.
- client_id: u32 stable xarray index in clients.
- no_kverbs_req: u8 — true if client tolerates !kverbs_provider devices.

REQ-4: struct ib_port_attr (per port_num, populated by ops->query_port):
- subnet_prefix: u64 (per-IBA prefix portion of GID).
- state: enum ib_port_state — NOP / DOWN / INIT / ARMED / ACTIVE / ACTIVE_DEFER.
- max_mtu / active_mtu: enum ib_mtu — IB_MTU_256 / _512 / _1024 / _2048 / _4096.
- phys_mtu: u32 — Ethernet/RoCE byte-MTU.
- gid_tbl_len: int — GID table length.
- ip_gids: 1-bit (RoCE devices using IP-derived GIDs).
- port_cap_flags: u32 (PortInfo CapabilityMask).
- max_msg_sz: u32.
- bad_pkey_cntr / qkey_viol_cntr: u32.
- pkey_tbl_len: u16.
- sm_lid / lid: u32 (24 bits in IBA, but stored in u32 for ext-LID).
- lmc: u8 (LID Mask Control).
- max_vl_num: u8.
- sm_sl / subnet_timeout / init_type_reply: u8.
- active_width: u8 (1x / 4x / 8x / 12x / 2x).
- active_speed: u16 (SDR / DDR / QDR / FDR / EDR / HDR / NDR / XDR).
- phys_state: u8 (enum ib_port_phys_state: SLEEP / POLLING / DISABLED / PORT_CFG_TRAINING / LINK_UP / LINK_ERR_RECOVERY / PHY_TEST).
- port_cap_flags2: u16.

REQ-5: struct ib_port_data (per-port runtime, allocated by alloc_port_data):
- ib_dev: back-pointer.
- immutable: struct ib_port_immutable (per-port: gid_tbl_len, pkey_tbl_len, core_cap_flags, max_mad_size) snapshot fixed at register-time.
- pkey_list_lock / pkey_list: per-port PKey cache.
- netdev_lock / netdev (RCU): per-port bound net_device (RoCE).
- netdev_tracker / ndev_hash_link: per-RCU hashtable bucket.
- cache: struct ib_port_cache (per-port GID + PKey table, mirrored from HW).
- port_counter: struct rdma_port_counter (per-port stats counter).
- sysfs: per-port struct ib_port (sysfs attribute group).

REQ-6: _ib_alloc_device(size, net) -> *ib_device:
- WARN_ON(size < sizeof(struct ib_device)); kzalloc(size, GFP_KERNEL).
- rdma_restrack_init(device).
- If ib_devices_shared_netns: force net = &init_net.
- rdma_init_coredev(&device->coredev, device, net).
- INIT_LIST_HEAD(event_handler_list), spin_lock_init(qp_open_list_lock), init_rwsem(event_handler_rwsem), mutex_init(unregistration_lock).
- xa_init_flags(client_data, XA_FLAGS_ALLOC).
- init_rwsem(client_data_rwsem); xa_init_flags(compat_devs, XA_FLAGS_ALLOC); mutex_init(compat_devs_mutex).
- init_completion(unreg_completion); INIT_WORK(unregistration_work, ib_unregister_work).
- spin_lock_init(cq_pools_lock); per-poll-ctx INIT_LIST_HEAD(cq_pools[i]).
- rwlock_init(cache_lock).
- device->uverbs_cmd_mask = BIT_ULL(IB_USER_VERBS_CMD_ALLOC_MW) | ALLOC_PD | ATTACH_MCAST | CLOSE_XRCD | CREATE_AH | CREATE_COMP_CHANNEL | CREATE_CQ | CREATE_QP | CREATE_SRQ | CREATE_XSRQ | DEALLOC_MW | DEALLOC_PD | DEREG_MR | DESTROY_AH | DESTROY_CQ | DESTROY_QP | DESTROY_SRQ | DETACH_MCAST | GET_CONTEXT | MODIFY_QP | MODIFY_SRQ | OPEN_QP | OPEN_XRCD | QUERY_DEVICE | QUERY_PORT | QUERY_QP | QUERY_SRQ | REG_MR | REREG_MR | RESIZE_CQ.
- mutex_init(subdev_lock); INIT_LIST_HEAD(subdev_list_head); INIT_LIST_HEAD(subdev_list).
- return device.

REQ-7: ib_register_device(device, name, dma_device) -> int:
- assign_name(device, name) — resolves "%d" → numeric ID via alloc_name + xarray reservation.
- WARN_ON(dma_device && !dma_device->dma_parms); device->dma_device = dma_device.
- setup_device(device): /* phase 1 */ ib_device_check_mandatory; populate device->attrs via ops->query_device; setup_port_data (alloc_port_data + verify_immutable for each port).
- ib_cache_setup_one(device) — populate GID/PKey caches.
- device->groups[0] = &ib_dev_attr_group; device->groups[1] = ops->device_group; ib_setup_device_attrs(device).
- ib_device_register_rdmacg(device) — register with RDMA cgroup controller.
- rdma_counter_init(device).
- dev_set_uevent_suppress(&device->dev, true); device_add(&device->dev).
- ib_setup_port_attrs(&device->coredev) — per-port sysfs.
- enable_device_and_get(device): /* phase 2 */ insert into xarray with DEVICE_REGISTERED mark; per-client add(); compat-dev shadowing if !ib_devices_shared_netns; allow further refs.
- On enable error: install prevent_dealloc_device shim, __ib_unregister_device, restore dealloc_driver, return.
- dev_set_uevent_suppress(&device->dev, false); ib_device_notify_register(device) — emit KOBJ_ADD uevent + netlink notification.
- ib_device_put(device); return 0.

REQ-8: ib_unregister_device(ib_dev):
- /* Synchronous tear-down. Caller must not hold a device ref. */
- mutex_lock(unregistration_lock); __ib_unregister_device(ib_dev); mutex_unlock.
- __ib_unregister_device: per-sub-device del; disable_device (clear DEVICE_REGISTERED, call per-client remove, wait until refcount==1); ib_device_unregister_rdmacg; ib_cache_cleanup_one; ib_teardown_port_attrs; device_del(&dev->dev); if ops->dealloc_driver: ops->dealloc_driver(device) (may free).

REQ-9: ib_unregister_device_and_put(ib_dev):
- Drop the caller's ref then unregister. Synchronizes with parallel unregister.

REQ-10: ib_unregister_device_queued(ib_dev):
- queue_work(ib_unreg_wq, &ib_dev->unregistration_work) — async path used from IRQ/atomic contexts (e.g., PCI EEH).

REQ-11: ib_unregister_driver(driver_id):
- down_read(devices_rwsem); for_each_marked DEVICE_REGISTERED with matching ops.driver_id: ib_device_get / queue unregister_work; up_read; flush_workqueue(ib_unreg_wq).

REQ-12: ib_register_client(client) -> int:
- refcount_set(&client->uses, 1); init_completion(&client->uses_zero).
- down_write(devices_rwsem) + down_write(clients_rwsem) — ordering critical.
- assign_client_id(client) — xa_alloc in clients with CLIENT_REGISTERED mark; client_id = highest_client_id++ (monotonic).
- for_each DEVICE_REGISTERED: add_client_context(device, client) (xa_store NULL in client_data with CLIENT_DATA_REGISTERED mark; client->add(device)).
- up_write(clients_rwsem) + up_write(devices_rwsem).
- On error: ib_unregister_client(client) cleanup; return.

REQ-13: ib_unregister_client(client):
- down_write(clients_rwsem); ib_client_put(client) — drop the registration ref so uses_zero can fire; xa_clear_mark(clients, client_id, CLIENT_REGISTERED); up_write.
- rcu_read_lock; xa_for_each(devices): if ib_device_try_get(device): rcu_read_unlock; remove_client_context(device, client_id) (calls client->remove if context registered); ib_device_put; rcu_read_lock; repeat.
- wait_for_completion(&client->uses_zero) — full fence: no callback in flight.
- remove_client_id(client) — xa_erase from clients.

REQ-14: ib_set_device_ops(dev, ops):
- For every recognized op pointer present in ops: if dev->ops field is NULL, splice it; else no-op (preserves caller's prior set).
- driver_id: WARN_ON if non-UNKNOWN being overwritten by a different non-UNKNOWN value; otherwise set.
- owner: WARN_ON if non-NULL being overwritten by a different non-NULL value; otherwise set.
- uverbs_abi_ver: overwrite if ops->uverbs_abi_ver is non-zero.
- uverbs_no_driver_id_binding |= ops->uverbs_no_driver_id_binding (sticky OR).
- uverbs_robust_udata |= ops->uverbs_robust_udata (sticky OR).
- SET_OBJ_SIZE(ib_ah, ib_counters, ib_cq, ib_dmah, ib_mw, ib_pd, ib_qp, ib_rwq_ind_table, ib_srq, ib_ucontext, ib_xrcd, rdma_counter).

REQ-15: SET_DEVICE_OP canonical list (exact upstream order, ib_set_device_ops): add_gid, add_sub_dev, advise_mr, alloc_dm, alloc_dmah, alloc_hw_device_stats, alloc_hw_port_stats, alloc_mr, alloc_mr_integrity, alloc_mw, alloc_pd, alloc_rdma_netdev, alloc_ucontext, alloc_xrcd, attach_mcast, check_mr_status, counter_alloc_stats, counter_bind_qp, counter_dealloc, counter_init, counter_unbind_qp, counter_update_stats, create_ah, create_counters, create_cq, create_user_cq, create_flow, create_qp, create_rwq_ind_table, create_srq, create_user_ah, create_wq, dealloc_dm, dealloc_dmah, dealloc_driver, dealloc_mw, dealloc_pd, dealloc_ucontext, dealloc_xrcd, del_gid, del_sub_dev, dereg_mr, destroy_ah, destroy_counters, destroy_cq, destroy_flow, destroy_flow_action, destroy_qp, destroy_rwq_ind_table, destroy_srq, destroy_wq, device_group, detach_mcast, disassociate_ucontext, drain_rq, drain_sq, enable_driver, fill_res_cm_id_entry, fill_res_cq_entry, fill_res_cq_entry_raw, fill_res_mr_entry, fill_res_mr_entry_raw, fill_res_qp_entry, fill_res_qp_entry_raw, fill_res_srq_entry, fill_res_srq_entry_raw, fill_stat_mr_entry, get_dev_fw_str, get_dma_mr, get_hw_stats, get_link_layer, get_netdev, get_numa_node, get_port_immutable, get_vf_config, get_vf_guid, get_vf_stats, iw_accept, iw_add_ref, iw_connect, iw_create_listen, iw_destroy_listen, iw_get_qp, iw_reject, iw_rem_ref, map_mr_sg, map_mr_sg_pi, mmap, mmap_get_pfns, mmap_free, modify_ah, modify_cq, modify_device, modify_hw_stat, modify_port, modify_qp, modify_srq, modify_wq, peek_cq, pgoff_to_mmap_entry, pre_destroy_cq, poll_cq, port_groups, post_destroy_cq, post_recv, post_send, post_srq_recv, process_mad, query_ah, query_device, query_gid, query_pkey, query_port, query_port_speed, query_qp, query_srq, query_ucontext, rdma_netdev_get_params, read_counters, reg_dm_mr, reg_user_mr, reg_user_mr_dmabuf, req_notify_cq, rereg_user_mr, resize_user_cq, set_vf_guid, set_vf_link_state, ufile_hw_cleanup, report_port_event.

REQ-16: ib_register_event_handler / ib_unregister_event_handler / ib_dispatch_event_clients:
- register: down_write(event_handler_rwsem); list_add_tail; up_write.
- unregister: down_write; list_del; up_write.
- dispatch_event_clients(event): down_read(event_handler_rwsem); list_for_each_entry(handler, list, list): handler->handler(handler, event); up_read.

REQ-17: ib_query_port(device, port_num, attr):
- if !rdma_is_port_valid(device, port_num): return -EINVAL.
- if rdma_protocol_iwarp(device, port_num): iw_query_port (synthesize from netdev).
- else: __ib_query_port → memset(attr, 0); ret = device->ops.query_port(device, port_num, attr); attr->subnet_prefix = ib_get_cached_subnet_prefix; ib_get_cached_lmc; etc.

REQ-18: ib_modify_device / ib_modify_port:
- ib_modify_device(dev, mask, modify): if ops->modify_device → call it (sys_image_guid + node_desc); else if mask==NODE_DESC: per-core fallback (memcpy node_desc). 
- ib_modify_port(dev, port_num, mask, modify): if ops->modify_port → call it; else -EOPNOTSUPP unless mask==CHANGE_CAP_MASK with no actual change (zero-mask shortcut).

REQ-19: Netns model:
- ib_devices_shared_netns (module param, default true): all devices visible in all netns.
- ib_devices_shared_netns=0: per-netns "compat" shadow devices via add_one_compat_dev / remove_one_compat_dev (one struct ib_core_device per (device, netns) pair).
- rdma_dev_change_netns: drop from current netns, attach to target netns (only allowed if device supports per-netns isolation).
- rdma_dev_access_netns(dev, net): true iff device's coredev.owner_net == net OR shared_netns OR device has compat entry for net.

REQ-20: Per-LSM policy change:
- ibdev_lsm_nb: register_lsm_notifier — on LSM_POLICY_CHANGE schedule ib_policy_change_work.
- ib_policy_change_task: for each registered device + per-port + per-PKey: revalidate uverbs ucontext security state via ib_security_modify_qp et al.

## Acceptance Criteria

- [ ] AC-1: ib_alloc_device returns a zeroed ib_device with refcount=1 and all locks/lists initialized; size >= sizeof(struct ib_device) WARNed otherwise.
- [ ] AC-2: ib_register_device populates kverbs_provider iff all 17 mandatory ops set; otherwise kverbs_provider=false but registration still succeeds for user-only consumers.
- [ ] AC-3: ib_register_device with name="mlx5_%d" expands to first free numeric ID.
- [ ] AC-4: After ib_register_device returns 0: /sys/class/infiniband/<name>/ exists, /sys/class/infiniband/<name>/ports/<n>/ exists for n in 1..phys_port_cnt, every registered ib_client.add was called.
- [ ] AC-5: ib_unregister_device calls every still-attached ib_client.remove before tearing down sysfs.
- [ ] AC-6: ib_register_client subsequently registered: add called once per existing registered device.
- [ ] AC-7: ib_unregister_client returns only after every client->remove has finished (uses_zero completion).
- [ ] AC-8: ib_set_device_ops twice with overlapping ops: second call does not overwrite already-set pointers (defensive splice semantics).
- [ ] AC-9: ib_query_port returns -EINVAL for port_num=0 (1-based ports) or > phys_port_cnt.
- [ ] AC-10: ib_dispatch_event_clients fans an IB_EVENT_PORT_ACTIVE to every registered handler under rwsem read lock.
- [ ] AC-11: ib_devices_shared_netns=0: a new netns sees only its compat shadow devices; init_net keeps the originals.
- [ ] AC-12: ib_device_get_by_netdev returns the device whose port netdev pointer matches; ib_device_put balances.
- [ ] AC-13: ib_device_rename rejects names colliding with an existing registered device.
- [ ] AC-14: An LSM policy change triggers re-evaluation: schedule_work(&ib_policy_change_work) is queued exactly once per change event.

## Architecture

```
struct IbDevice {
  dev: Device,                         // embedded driver-model device
  ops: IbDeviceOps,                    // ~150-entry vtable
  name: String,
  index: u32,                          // xarray slot
  node_type: u8,                       // RDMA_NODE_*
  node_guid: u64_be,
  sys_image_guid: u64_be,
  attrs: IbDeviceAttr,
  phys_port_cnt: u32,
  num_comp_vectors: u32,
  port_data: Box<[IbPortData]>,        // index 0..phys_port_cnt-1
  coredev: IbCoreDevice,               // per-netns shadow anchor
  compat_devs: XArray<IbCoreDevice>,
  client_data: XArray<*mut c_void>,    // per-client priv, CLIENT_DATA_REGISTERED mark
  client_data_rwsem: RwSemaphore,
  event_handler_list: ListHead,
  event_handler_rwsem: RwSemaphore,
  unregistration_lock: Mutex,
  unreg_completion: Completion,
  unregistration_work: Work,
  refcount: RefCount,
  cache_lock: RwLock,
  cq_pools: [ListHead; IB_POLL_LAST_POOL_TYPE + 1],
  cq_pools_lock: SpinLock,
  subdev_list / subdev_list_head: ListHead,
  subdev_lock: Mutex,
  uverbs_cmd_mask: u64,                // bitmap of supported user-verb commands
  kverbs_provider: bool,
  groups: [Option<*const AttributeGroup>; 4],
  dma_device: Option<*mut Device>,
}

struct IbPortAttr {
  subnet_prefix: u64,
  state: u8,                           // enum ib_port_state
  max_mtu: u8, active_mtu: u8,
  phys_mtu: u32,
  gid_tbl_len: i32,
  ip_gids: bool,
  port_cap_flags: u32,
  max_msg_sz: u32,
  bad_pkey_cntr: u32, qkey_viol_cntr: u32,
  pkey_tbl_len: u16,
  sm_lid: u32, lid: u32,
  lmc: u8, max_vl_num: u8, sm_sl: u8, subnet_timeout: u8, init_type_reply: u8,
  active_width: u8, active_speed: u16,
  phys_state: u8,
  port_cap_flags2: u16,
}

struct IbClient {
  name: &'static str,
  add: fn(&mut IbDevice) -> Result<()>,
  remove: fn(&mut IbDevice, *mut c_void),
  rename: Option<fn(&mut IbDevice, *mut c_void)>,
  get_nl_info: Option<fn(...)>,
  get_global_nl_info: Option<fn(...)>,
  get_net_dev_by_params: Option<fn(...) -> *mut NetDevice>,
  uses: RefCount,
  uses_zero: Completion,
  client_id: u32,
  no_kverbs_req: bool,
}
```

`IbDevice::alloc(size, net) -> *mut IbDevice`:
1. WARN_ON(size < size_of::<IbDevice>()).
2. dev = kzalloc(size, GFP_KERNEL); on NULL return NULL.
3. rdma_restrack_init(dev) — on error: kfree; return NULL.
4. net = if ib_devices_shared_netns { &init_net } else { net }.
5. rdma_init_coredev(&dev.coredev, dev, net).
6. INIT_LIST_HEAD on event_handler_list; init_rwsem on event_handler_rwsem.
7. spin_lock_init on qp_open_list_lock; mutex_init on unregistration_lock.
8. xa_init_flags(&dev.client_data, XA_FLAGS_ALLOC); init_rwsem(&dev.client_data_rwsem).
9. xa_init_flags(&dev.compat_devs, XA_FLAGS_ALLOC); mutex_init(&dev.compat_devs_mutex).
10. init_completion(&dev.unreg_completion); INIT_WORK(&dev.unregistration_work, ib_unregister_work).
11. spin_lock_init(&dev.cq_pools_lock); for i in 0..ARRAY_SIZE(cq_pools): INIT_LIST_HEAD(&dev.cq_pools[i]).
12. rwlock_init(&dev.cache_lock).
13. Populate dev.uverbs_cmd_mask = (per-REQ-6 bitmap).
14. mutex_init(&dev.subdev_lock); INIT_LIST_HEAD(&dev.subdev_list_head); INIT_LIST_HEAD(&dev.subdev_list).
15. return dev.

`IbDevice::register(dev, name, dma_device) -> Result<()>`:
1. assign_name(dev, name) — propagate -EEXIST / -ENOMEM.
2. WARN_ON(dma_device && !dma_device.dma_parms); dev.dma_device = dma_device.
3. setup_device(dev): ib_device_check_mandatory; ret = ops.query_device(dev, &dev.attrs); setup_port_data (per-port alloc_port_data + verify_immutable).
4. ib_cache_setup_one(dev) — populate GID/PKey caches.
5. dev.groups[0] = &ib_dev_attr_group; dev.groups[1] = ops.device_group; ib_setup_device_attrs(dev).
6. ib_device_register_rdmacg(dev); rdma_counter_init(dev).
7. dev_set_uevent_suppress(&dev.dev, true); device_add(&dev.dev).
8. ib_setup_port_attrs(&dev.coredev).
9. enable_device_and_get(dev): /* xarray DEVICE_REGISTERED + per-client add */.
10. On enable error: install prevent_dealloc shim, ib_device_put, __ib_unregister_device, restore dealloc_driver, return.
11. dev_set_uevent_suppress(&dev.dev, false); ib_device_notify_register(dev) — KOBJ_ADD + netlink.
12. ib_device_put(dev); return Ok.

`IbDevice::unregister(ib_dev)`:
1. mutex_lock(&ib_dev.unregistration_lock).
2. __ib_unregister_device(ib_dev): for each sub-device: ib_del_sub_device_and_put; disable_device (clear DEVICE_REGISTERED mark; per-client remove_client_context; wait refcount==1); ib_device_unregister_rdmacg; ib_cache_cleanup_one; ib_teardown_port_attrs; device_del; if ops.dealloc_driver: ops.dealloc_driver(ib_dev).
3. mutex_unlock(&ib_dev.unregistration_lock).

`IbClient::register(client) -> Result<()>`:
1. refcount_set(&client.uses, 1); init_completion(&client.uses_zero).
2. down_write(&devices_rwsem); down_write(&clients_rwsem).
3. assign_client_id(client) — xa_alloc(&clients, &client.client_id, client, mark CLIENT_REGISTERED); highest_client_id = max + 1.
4. For each xa_marked DEVICE_REGISTERED in devices: add_client_context(device, client) → xa_store(&device.client_data, client.client_id, NULL, mark CLIENT_DATA_REGISTERED); client.add(device).
5. up_write(&clients_rwsem); up_write(&devices_rwsem).
6. On error: ib_unregister_client(client); return err.

`IbClient::unregister(client)`:
1. down_write(&clients_rwsem); ib_client_put(client) — drop registration ref; xa_clear_mark(&clients, client.client_id, CLIENT_REGISTERED); up_write.
2. rcu_read_lock; xa_for_each(&devices, device): if ib_device_try_get(device): rcu_read_unlock; remove_client_context(device, client.client_id) — under client_data_rwsem write: if xa_marked CLIENT_DATA_REGISTERED: client.remove(device, client_data); xa_erase; xa_clear_mark; ib_device_put; rcu_read_lock; continue.
3. wait_for_completion(&client.uses_zero) — full fence: returns only after all per-device removes complete.
4. remove_client_id(client) — xa_erase(&clients, client.client_id).

`IbDevice::set_ops(dev, ops)`:
1. dev_ops = &dev.ops.
2. If ops.driver_id != RDMA_DRIVER_UNKNOWN: WARN if existing driver_id is set to a different non-UNKNOWN value; dev_ops.driver_id = ops.driver_id.
3. If ops.owner: WARN if existing owner is non-NULL and differs; dev_ops.owner = ops.owner.
4. If ops.uverbs_abi_ver: dev_ops.uverbs_abi_ver = ops.uverbs_abi_ver.
5. dev_ops.uverbs_no_driver_id_binding |= ops.uverbs_no_driver_id_binding.
6. dev_ops.uverbs_robust_udata |= ops.uverbs_robust_udata.
7. For each canonical op in REQ-15: if ops.<name>: if !dev_ops.<name>: dev_ops.<name> = ops.<name>.
8. For each canonical size in {ib_ah, ib_counters, ib_cq, ib_dmah, ib_mw, ib_pd, ib_qp, ib_rwq_ind_table, ib_srq, ib_ucontext, ib_xrcd, rdma_counter}: if ops.size_<name>: if !dev_ops.size_<name>: dev_ops.size_<name> = ops.size_<name>.

`IbDevice::query_port(dev, port_num, attr) -> Result<()>`:
1. if !rdma_is_port_valid(dev, port_num): return Err(EINVAL).
2. attr.zero().
3. if rdma_protocol_iwarp(dev, port_num): return iw_query_port(dev, port_num, attr).
4. ret = dev.ops.query_port(dev, port_num, attr); if ret: return Err.
5. attr.subnet_prefix = ib_get_cached_subnet_prefix(dev, port_num).
6. attr.lmc = ib_get_cached_lmc(dev, port_num).
7. attr.port_cap_flags2 = ib_get_cached_port_cap_flags2(dev, port_num).
8. return Ok.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `alloc_size_check` | INVARIANT | per-_ib_alloc_device: size >= sizeof(IbDevice). |
| `register_phase_ordering` | INVARIANT | setup_device < ib_cache_setup_one < device_add < ib_setup_port_attrs < enable_device_and_get < notify_register. |
| `client_lock_ordering` | INVARIANT | devices_rwsem acquired before clients_rwsem. |
| `client_uses_zero_completes` | INVARIANT | per-unregister_client: returns only after uses → 0. |
| `set_ops_no_overwrite` | INVARIANT | per-set_device_ops: pre-set non-NULL pointers preserved. |
| `port_num_one_based` | INVARIANT | per-query_port: port_num in 1..=phys_port_cnt. |
| `mandatory_ops_or_no_kverbs` | INVARIANT | per-register: missing-mandatory ⟹ kverbs_provider=false. |
| `compat_dev_iff_unshared_netns` | INVARIANT | per-add_one_compat_dev: invoked only when !ib_devices_shared_netns. |
| `event_handler_rwsem_held_on_dispatch` | INVARIANT | per-dispatch_event_clients: read-held during fanout. |
| `refcount_balanced` | INVARIANT | every ib_device_get / ib_device_try_get balanced with ib_device_put. |

### Layer 2: TLA+

`drivers/infiniband/core/device.tla`:
- States: allocated → name-assigned → setup → cache-up → groups-set → device-added → port-attrs-up → enabled → registered → disabling → disabled → dealloc.
- Per-client lifecycle: unregistered → registered → per-device-added → per-device-removed → unregistered.
- Properties:
  - `safety_no_op_pointer_overwrite` — per-set_ops: existing non-NULL fields stay.
  - `safety_remove_before_dealloc` — per-unreg: all client_data slots cleared before device_del.
  - `safety_client_remove_in_flight_blocks_unregister` — uses_zero ⟹ unregister waits.
  - `safety_compat_devs_only_under_unshared_netns` — compat_devs.populated ⟹ !ib_devices_shared_netns.
  - `safety_mandatory_iff_kverbs_provider` — kverbs_provider ⟺ all 17 mandatory ops set.
  - `liveness_register_eventually_completes` — register: terminates in success or returns err.
  - `liveness_unregister_eventually_completes` — unregister: terminates when refs == 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IbDevice::alloc` post: refcount==1 ∧ all lists/locks init'd | `IbDevice::alloc` |
| `IbDevice::register` post: on Ok device in xarray with DEVICE_REGISTERED mark ∧ sysfs created ∧ all clients added | `IbDevice::register` |
| `IbDevice::unregister` post: device removed from xarray ∧ all clients removed ∧ sysfs torn down | `IbDevice::unregister` |
| `IbClient::register` post: every existing DEVICE_REGISTERED device has add called | `IbClient::register` |
| `IbClient::unregister` post: every existing attachment has remove called ∧ uses==0 | `IbClient::unregister` |
| `IbDevice::set_ops` post: ∀ field f. dev.ops.f' == if dev.ops.f.is_some() { dev.ops.f } else { ops.f } | `IbDevice::set_ops` |
| `IbDevice::query_port` post: port_num valid ⟹ attr fully populated; invalid ⟹ EINVAL | `IbDevice::query_port` |
| `IbDevice::dispatch_event_clients` post: event delivered to every list_for_each_entry under read-rwsem | `IbDevice::dispatch_event_clients` |

### Layer 4: Verus/Creusot functional

`Per-driver alloc_device → set_ops → register_device → (per-client add) → query_port / modify_port / register_event_handler → dispatch_event → (per-client remove) → unregister_device → dealloc_device` semantic equivalence: per-Documentation/infiniband/core_locking.rst, Documentation/infiniband/user_verbs.rst, IBA Architecture Specification Vol. 1 § 11.4 (device management), § 16.1 (subnet management). Per-vtable layout and EXPORT_SYMBOL set must be byte-identical against upstream for module-binary compat with out-of-tree drivers.

## Hardening

(Inherits row-1 features from `drivers/infiniband/00-overview.md` § Hardening.)

Device-registration reinforcement:

- **Per-mandatory-ops gate** — `ib_device_check_mandatory` populates `kverbs_provider`; missing ops surface kverbs_provider=false but never silently null-deref a function pointer.
- **Per-set_device_ops splice semantics (no overwrite)** — defense against driver-bug op-stomping.
- **Per-devices_rwsem before clients_rwsem ordering** — defense against per-AB-BA deadlock with parallel register_client / register_device.
- **Per-CLIENT_DATA_REGISTERED xarray mark distinct from NULL-stored** — defense against per-client-NULL-data ambiguity (NULL is legal client data).
- **Per-uses_zero completion fence on unregister_client** — defense against per-remove-callback-still-running UAF.
- **Per-prevent_dealloc_device shim on register error** — defense against per-double-free under racing unregister.
- **Per-rdma_restrack_init on alloc** — defense against per-resource-leak (every QP/CQ/MR tracked).
- **Per-compat-dev per-netns isolation** — defense against per-cross-netns device visibility leak when ib_devices_shared_netns=0.
- **Per-LSM-policy-change re-evaluation hook** — defense against per-stale-context post-policy-change.
- **Per-name uniqueness via alloc_name + xarray** — defense against per-duplicate-name registration corrupting /sys/class/infiniband.
- **Per-RDMA-cgroup registration before xarray-enable** — defense against per-resource-quota bypass for in-flight allocations.
- **Per-port immutable snapshot at register** — defense against per-port-cnt or per-pkey_tbl_len change while clients hold cached pointers.
- **Per-RCU-protected netdev lookup** — defense against per-netdev-unregister race in ib_device_get_by_netdev.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `ib_device`, `ib_client`, and `ib_port_data`; sysfs and netlink boundary buffers (`RDMA_NL_*`) strictly bounded before copy-in/out.
- **PAX_KERNEXEC** — IB device core in W^X kernel text; `ib_device_ops`, per-client tables, and the immutable per-port snapshot live in `__ro_after_init` memory.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `ib_register_device`, `ib_register_client`, and rdmacg cgroup-association entry points.
- **PAX_REFCOUNT** — saturating `refcount_t` on `ib_device`, `ib_client`, and per-port `ib_port_data`; xarray-based ID tables verified against ref overflow.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `ib_device` and per-port snapshot slabs so stale GID/PKey tables cannot leak across re-registration.
- **PAX_UDEREF** — SMAP/PAN enforced on every RDMA netlink and sysfs entry into the device core.
- **PAX_RAP / kCFI** — `ib_device_ops` (`add`, `remove`, `query_device`, `query_port`, `get_netdev`) and `ib_client` ops tables `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and `ib_device->dev` pointer disclosure behind CAP_SYSLOG; suppress `%p` printks on per-port data and netdev pointers.
- **GRKERNSEC_DMESG** — restrict device-register/unregister, port-state, and rdmacg-failure banners to CAP_SYSLOG.
- **ucontext CAP_NET_ADMIN gate** — `ib_uverbs_get_context` and any user-context allocation require CAP_NET_ADMIN in the device's net namespace (in addition to file-mode policy on `/dev/infiniband/*`).
- **kverbs allowlist** — kernel-only verbs entry points are reachable only by ULPs registered through `ib_register_client`; userspace-reachable verbs strictly disjoint from kverbs allowlist.
- **rdmacg ordering invariant** — RDMA cgroup registration completes before xarray enables device visibility so no in-flight resource allocation escapes quota enforcement.
- **Port-immutable snapshot** — `port_immutable` taken at register time and never mutated; defense against attribute-mutation races against cached client pointers.
- **RCU-protected netdev lookup** — `ib_device_get_by_netdev` only traverses RCU-grace-period-safe pointers, defeating netdev-unregister races.
- **net-namespace isolation** — every `ib_device` carries an owning net-ns; userland `RDMA_NL_*` ops refuse to operate across foreign namespaces.

Rationale: the IB device core is the registration root for every QP/CQ/MR allocator and exposes a netlink/sysfs surface that can synthesise userspace contexts capable of pinning kernel DMA-mapped memory. Without CAP_NET_ADMIN gating on ucontext, kverbs/userverbs disjointness, RAP/kCFI on `ib_device_ops`, and refcount-overflow trapping on devices and clients, a single in-flight unregister race becomes a kernel pointer disclosure and an unbounded DMA-pinning DoS.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/infiniband/core/verbs.c` verbs entry points (covered in `verbs.md` Tier-3)
- `drivers/infiniband/core/sysfs.c` sysfs attribute groups + per-port attrs (covered separately if expanded)
- `drivers/infiniband/core/uverbs_*.c` uverbs char-dev + ABI (covered separately if expanded)
- `drivers/infiniband/core/cache.c` GID/PKey cache implementation (covered separately if expanded)
- `drivers/infiniband/core/cma.c` rdma_cm (covered separately if expanded)
- `drivers/infiniband/core/mad.c` Management Datagram (covered separately if expanded)
- `drivers/infiniband/core/cm.c` Communication Manager (covered separately if expanded)
- Per-driver HCA implementations (mlx5, hfi1, irdma, bnxt_re, ...) (each is its own Tier-3 if expanded)
- Implementation code
