# Tier-3: drivers/net/ethernet/mellanox/mlx5/core/en_*.c — mlx5 en netdev path (RX/TX/main + XDP + AF_XDP + GRO/GSO + flow steering integration)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
upstream-paths:
  - drivers/net/ethernet/mellanox/mlx5/core/en_main.c
  - drivers/net/ethernet/mellanox/mlx5/core/en_rx.c
  - drivers/net/ethernet/mellanox/mlx5/core/en_tx.c
  - drivers/net/ethernet/mellanox/mlx5/core/en_common.c
  - drivers/net/ethernet/mellanox/mlx5/core/en_dim.c
  - drivers/net/ethernet/mellanox/mlx5/core/en_arfs.c
  - drivers/net/ethernet/mellanox/mlx5/core/en_ethtool.c
  - drivers/net/ethernet/mellanox/mlx5/core/en/...
  - include/linux/mlx5/...
-->

## Summary

The "en" sub-driver of mlx5 — the netdev path for Mellanox/NVIDIA ConnectX-4/5/6/7/8 NICs and BlueField DPU host-side. Most-deployed datacenter NIC driver: every cloud (AWS/Azure/GCP/OCI), every HPC cluster, every Mellanox/NVIDIA-enabled bare-metal server, every BlueField host PCIe Ethernet has this driver. Implements: per-channel RX queue (RQ) + TX queue (SQ) + completion queue (CQ) tied to one NAPI per CPU, mlx5-firmware-mediated flow steering for hardware offload of MAC/VLAN/L2/L3/L4 filtering + RSS hash distribution, ASIC-managed page-pool integration for zero-alloc RX, full XDP-DRV native + XDP_REDIRECT + XDP_TX support, AF_XDP zerocopy, GRO/GSO, TSO offload, RX/TX checksum offload, IPSec inline crypto offload, TLS-TX/RX device offload, FCoE-FCS handling, PTP hardware timestamping, dynamic interrupt moderation (DIM), aRFS (accelerated RFS for per-flow MSI-X-vector pinning), per-queue stats via ethtool.

This Tier-3 covers `drivers/net/ethernet/mellanox/mlx5/core/en_*.c` (~10K lines spread across en_main.c, en_rx.c, en_tx.c, en_common.c, en_dim.c, en_arfs.c, en_ethtool.c, plus subdir `en/{params,health,fs,...}`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mlx5e_priv` | per-netdev priv state | `kernel::mlx5::en::Priv` |
| `struct mlx5e_channel` | per-channel (NAPI + RQ + SQ + CQ tied together) | `kernel::mlx5::en::Channel` |
| `struct mlx5e_rq` | per-channel RX queue | `kernel::mlx5::en::Rq` |
| `struct mlx5e_sq` | per-channel TX queue | `kernel::mlx5::en::Sq` |
| `struct mlx5e_cq` | per-channel completion queue | `kernel::mlx5::en::Cq` |
| `struct mlx5e_xdpsq` | per-channel XDP TX queue | `kernel::mlx5::en::XdpSq` |
| `mlx5e_init_dev(netdev)` / `_cleanup_dev(...)` | netdev init/cleanup | `Priv::init_dev` / `_cleanup_dev` |
| `mlx5e_open(netdev)` / `_close(netdev)` | netdev open/close | `Priv::open` / `_close` |
| `mlx5e_open_channels(...)` / `_close_channels(...)` | per-channel batch open/close | `Priv::open_channels` / `_close_channels` |
| `mlx5e_open_channel(c, params, &cparam)` / `_close_channel(c)` | per-channel open/close | `Channel::open` / `_close` |
| `mlx5e_open_rq(...)` / `_close_rq(rq)` | per-RQ open/close | `Rq::open` / `_close` |
| `mlx5e_open_sq(...)` / `_close_sq(sq)` | per-SQ open/close | `Sq::open` / `_close` |
| `mlx5e_alloc_rx_wqes(rq, ix, wqe_bulk)` | RX descriptor pre-fill | `Rq::alloc_rx_wqes` |
| `mlx5e_handle_rx_cqe_*` family | per-RX-completion handler (per-WQ-format variant) | `Rq::handle_rx_cqe_*` |
| `mlx5e_xmit(skb, netdev)` | netdev_ops::ndo_start_xmit | `Priv::xmit` |
| `mlx5e_sq_xmit_simple(...)` / `_sq_xmit_wqe(...)` | TX WQE construction | `Sq::xmit_*` |
| `mlx5e_napi_poll(napi, budget)` | NAPI poll callback | `Channel::napi_poll` |
| `mlx5e_poll_rx_cq(cq, budget)` / `_poll_tx_cq(cq, napi_budget)` | per-CQ poll | `Cq::poll_rx_cq` / `_poll_tx_cq` |
| `mlx5e_xdp_xmit(...)` / `_xdp_redirect_flush(...)` | XDP_REDIRECT path | `Priv::xdp_xmit` / `_xdp_redirect_flush` |
| `mlx5e_xdp_handle(rq, di, ...)` | XDP prog dispatch | `Rq::xdp_handle` |
| `mlx5e_xsk_*` family | AF_XDP zerocopy path | `Priv::xsk_*` |
| `mlx5e_create_indirect_rqts(priv, ...)` | RSS indirection table | `Priv::create_indirect_rqts` |
| `mlx5e_create_tises(priv, ...)` | per-priority TIS (Transport Interface Send) | `Priv::create_tises` |
| `mlx5e_set_rx_mode_core(...)` | promiscuous / allmulti / mc-filter | `Priv::set_rx_mode_core` |
| `mlx5e_arfs_create_tables(priv)` / `_arfs_destroy_tables(priv)` | aRFS tables | `Priv::arfs_create_tables` |
| `mlx5e_open_xdpsq(c, ...)` / `_close_xdpsq(...)` | XDP TX SQ | `XdpSq::open` / `_close` |
| `mlx5e_dim_work(work)` | DIM (Dynamic Interrupt Moderation) work | `Channel::dim_work` |
| `mlx5e_modify_rq_state(rq, curr, new_state)` / `_modify_sq_state(...)` / `_modify_cq_state(...)` | per-WQ state transition (RST → READY → ERR) via firmware cmd | `Rq::modify_state` / etc. |

## Compatibility contract

REQ-1: Per-NIC PCI ID match table covers ConnectX-4 / ConnectX-5 / ConnectX-6 (Dx, Lx, EN, OCP, BlueField-1/2/3 host PCIe) / ConnectX-7 / ConnectX-8 + every per-vendor VF.

REQ-2: Per-channel structure: 1 NAPI per CPU (up to channels-count from CC_ETH_PARAMS_AC limit); each channel owns 1 RQ + 1+ SQ (one per traffic-class) + 1 CQ + 1+ XDP-SQ.

REQ-3: RQ formats: legacy (per-WQE = single RX descriptor), striding-RQ (per-WQE = 4-256 strides each holding one packet — much more cache-efficient for small-packet workloads).

REQ-4: TX formats: linear-skb-data (single WQE), GSO/TSO (per-skb fragment list packed into multi-WQE chain), inline-data (small-packet inline up to 256 bytes for low-latency).

REQ-5: XDP support: native (XDP-DRV) on every channel; per-channel XDP-SQ pre-allocated for XDP_TX path; XDP_REDIRECT routed via xdp_do_redirect → cross-ifindex devmap/cpumap.

REQ-6: AF_XDP zerocopy: per-RQ + per-SQ switched to XSK mode when bound; UMEM frame-ring acts as RX descriptor source AND TX completion destination.

REQ-7: RSS via per-tirs (Transport Interface Receive); indirection table maps hash buckets to per-RQ; supports XOR / Toeplitz hash, IPv4 / IPv6 / TCP / UDP src+dst hash.

REQ-8: aRFS (Accelerated RFS): per-flow ASIC-side filter steers packet to specific RX queue / CPU based on `rfs_table`; integrates with kernel's RFS rfs_filter.

REQ-9: TLS-TX device offload: when `setsockopt(SOL_TLS, TLS_TX, &cipher)` succeeds, mlx5 ASIC handles per-record encryption + crypto-state per-flow.

REQ-10: TLS-RX device offload: same for decryption.

REQ-11: IPSec inline + crypto offload (CONFIG_MLX5_EN_IPSEC): per-XFRM-state offload via `xfrm_state_add(.., XFRMA_OFFLOAD_DEV)`.

REQ-12: PTP hardware timestamping: per-NIC PHC (PTP Hardware Clock) registered via `ptp_clock_register`; `/dev/ptp<N>` chardev appears; phc2sys synchronizes.

REQ-13: DIM (Dynamic Interrupt Moderation): per-channel adaptive RX coalescing — adjusts `(timer, packet_count)` thresholds based on observed throughput + latency; integrates with `lib/dim/`.

REQ-14: ndo_xdp_features: BIT(NETDEV_XDP_ACT_BASIC) | BIT(NDA_XDP_ACT_REDIRECT) | BIT(NETDEV_XDP_ACT_XSK_ZEROCOPY) | BIT(NETDEV_XDP_ACT_RX_SG) | BIT(NETDEV_XDP_ACT_NDO_XMIT_SG); ethtool `xdp-features` reflects.

REQ-15: ethtool ops: per-NIC + per-RQ + per-SQ statistics (~600+ counters); rxq + txq config; coalesce config; RSS config; ringparam config; pauseparam; channel-count; linkksettings; flow-classify; reset; firmware-flash via devlink.

## Acceptance Criteria

- [ ] AC-1: mlx5_core probes ConnectX-6 100GbE on reference HW; `ip link show ens0f0` shows operstate UP.
- [ ] AC-2: iperf3 over mlx5 100GbE achieves > 95 Gbps line-rate.
- [ ] AC-3: XDP_PASS test: `xdp-loader load -m native ens0f0 xdp_pass.o` loads; packets pass through.
- [ ] AC-4: XDP_REDIRECT cross-ifindex: redirect from ens0f0 to ens0f1 via devmap; pps within 10% of physical line-rate.
- [ ] AC-5: AF_XDP zerocopy: `xdpsock --rx --tx -i ens0f0 --zero-copy` achieves > 50 Mpps at small-packet.
- [ ] AC-6: TLS-TX offload: nginx with kTLS + mlx5 TLS-TX offload serves HTTPS at line-rate; per-conn encryption visible offloaded via `ethtool -S` counters.
- [ ] AC-7: PTP test: `phc2sys -s ens0f0 -O 0 -m` synchronizes system clock to PHC; offset < 1µs.
- [ ] AC-8: SR-IOV test: `echo 8 > /sys/bus/pci/devices/.../sriov_numvfs` creates 8 VFs; each enumerates with mlx5_core_vf driver.
- [ ] AC-9: Stress: sustained 100GbE throughput for 1 hour; KASAN clean; no urb leak; no skb leak.
- [ ] AC-10: kselftest mlx5-related tests pass.

## Architecture

`Priv` lives in `kernel::mlx5::en::Priv`:

```
struct Priv {
  netdev: Arc<NetDevice>,
  mdev: Arc<MlxCoreDevice>,
  refcount: Refcount,
  state: AtomicI32,           // STATE_DESTROYING / OPENED / OPENING / etc.
  channels: Mutex<KBox<Channels>>,
  q_counter: u32,
  drop_rq_q_counter: u32,
  fs: KBox<FlowSteering>,
  arfs_tables: Option<KBox<ArfsTables>>,
  tunnel_indir_tirs: Option<KBox<TunnelTirs>>,
  rss_params: KBox<RssParams>,
  channel_stats: PerCpu<KBox<ChannelStats>>,
  ptp_state: AtomicI32,
  hwstamp_config: HwTstampConfig,
  prof: Arc<Profile>,
  ipsec: Option<Arc<IpsecOffload>>,
  tls: Option<Arc<TlsOffload>>,
  ...
}

struct Channel {
  napi: Napi,
  rq: KBox<Rq>,
  sq: ArrayMutex<MAX_TC_NUM, KBox<Sq>>,
  xsk_sq: Option<KBox<Sq>>,
  xdp_sq: KBox<XdpSq>,
  cq: KBox<Cq>,
  xsk_cq: Option<KBox<Cq>>,
  xdp_cq: KBox<Cq>,
  ix: u32,                     // channel index
  tirn: u32,
  cpu: i32,
  netdev: Arc<NetDevice>,
  priv_: Arc<Priv>,
  ...
}

struct Rq {
  base: WqState,
  wqe_type: u8,                // RQ_WQE_TYPE_LINKED_LIST / _CYCLIC / _STRIDING_RQ
  page_pool: Arc<PagePool>,    // per-RQ page-pool
  cq: NonNull<Cq>,
  xdp_prog: AtomicPtr<BpfProg>, // attached XDP prog
  xsk_pool: Option<Arc<XskPool>>,
  channel: NonNull<Channel>,
  alloc_count: AtomicU64,
  hw_mtu: u32,
  ...
}
```

Probe path (auxiliary-bus):
1. mlx5_core (parent) creates aux-device for "en" sub-driver.
2. mlx5e_probe(aux_dev): allocate `Priv` + register netdev.
3. `mlx5e_init_dev(netdev)`: populate netdev_ops + ethtool_ops + xdp_ops.

Open path `Priv::open(netdev)`:
1. Configure firmware: per-NIC profile-init (default profile vs custom).
2. `mlx5e_open_channels(priv, &priv.channels.params)`:
   - Per-CPU: `Channel::open(&channel, &cparam)`:
     - Allocate CQ via firmware `CMD_CREATE_CQ`.
     - Allocate RQ via `CMD_CREATE_RQ` with WQE format choice (legacy / striding).
     - For each TC: allocate SQ via `CMD_CREATE_SQ` + bind to TC.
     - Allocate XDP-SQ.
     - Set up per-channel page-pool.
     - Pre-fill RQ via `mlx5e_alloc_rx_wqes`.
     - Open NAPI: `napi_enable(&channel.napi)`.
3. Configure RSS: `mlx5e_create_indirect_rqts` + `mlx5e_create_inner_ttc_table` + per-protocol-class steering rules.
4. Set link state UP.

NAPI poll `Channel::napi_poll(napi, budget)`:
1. `Cq::poll_tx_cq(&channel.cq, budget)` first → free completed TX skbs.
2. `Cq::poll_rx_cq(&channel.cq, budget - tx_completed)` → process RX completions.
3. If processed < budget: napi_complete_done; rearm IRQ.
4. Else: return budget; NAPI re-schedules.

RX completion path `Rq::handle_rx_cqe`:
1. Per-CQE: `wqe_index = cqe.wqe_counter`.
2. Look up `di = rq.fqes[wqe_index]` → page-pool fragment.
3. Build skb OR xdp_buff.
4. If `rq.xdp_prog`: `Rq::xdp_handle(rq, di, ...)`:
   - `bpf_prog_run_xdp(prog, &xdp)` → action.
   - PASS: convert to skb + GRO/normal receive.
   - DROP: page-pool put_page.
   - TX: re-enqueue on `channel.xdp_sq`.
   - REDIRECT: `xdp_do_redirect(...)`.
5. Else (no XDP): `napi_gro_receive(napi, skb)`.
6. Increment per-CPU stats.

TX path `Priv::xmit(skb, netdev)`:
1. Select `txq = netdev_pick_tx(skb)` → per-CPU TX queue.
2. `sq = priv.channels.c[cpu].sq[tc]`.
3. `Sq::xmit_simple(sq, skb)`:
   - Build TX WQE: header + control + ETH segment + DATA segment(s).
   - For TSO: split skb into LSO segments.
   - For inline-data (small packet): inline up to 256B in WQE.
   - For DMA: dma_map_skb → fill DATA-segments with DMA addrs.
   - Increment `sq.pc` (producer counter).
   - Ring doorbell via `mlx5e_notify_hw(sq, ...)`.

XDP_REDIRECT flush `Priv::xdp_redirect_flush`:
- Per-channel `xdp_redirect_xmit_xdp_buff_*` flushes pending XDP redirect frames at end of NAPI poll.

DIM `Channel::dim_work`:
- Periodically (per-channel): read RX-cycle + RX-pkt counters → compute new (timer, count) thresholds via `dim_calc_stats`.
- Apply via `mlx5e_modify_cq_period_mode(cq, new_mode)` firmware command.

aRFS: per-flow filter installed via `mlx5e_arfs_table_insert` → ASIC steers matching flow to per-CPU RX queue; ASIC NIC RX-queue ↔ NAPI ↔ CPU ↔ application thread alignment for cache-locality.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `priv_no_uaf` | UAF | `Arc<Priv>` outlives all `Channel` instances; close waits for each channel's NAPI to complete. |
| `rq_pp_no_uaf` | UAF | per-RQ page-pool drained before RQ destroy; per-page refcount tracked. |
| `wqe_idx_no_oob` | OOB | per-WQ wqe_index bounded by wq_size; `pc - cc` fits within ring. |
| `xdp_buff_no_oob` | OOB | xdp_buff data + data_end bounds checked against page-frag; XDP prog bounded mutations validated. |

### Layer 2: TLA+

`models/net-eth/page_pool.tla` (parent-declared): proves per-NAPI page-pool.
`models/net-eth/napi_budget.tla` (parent-declared): proves NAPI poll-budget.
`models/net-eth/af_xdp_umem.tla` (parent-declared): proves AF_XDP UMEM.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Channel::open` post: napi enabled; RQ pre-filled; SQ ready; CQ armed | `Channel::open` |
| `Channel::napi_poll` post: 0 ≤ processed ≤ budget; if processed == budget, IRQ NOT re-armed (NAPI re-schedule) | `Channel::napi_poll` |
| `Sq::xmit_simple` post: skb either consumed (returned NETDEV_TX_OK) OR SQ full (NETDEV_TX_BUSY) — never UAF | `Sq::xmit_simple` |
| RSS: hash bucket → per-RQ mapping consistent across packets in same flow | `Priv::create_indirect_rqts` |

### Layer 4: Verus/Creusot functional

`Priv::open → Channel::napi_poll → Rq::handle_rx_cqe → napi_gro_receive` round-trip equivalence: every packet received by NIC and acked by HW becomes either skb on netif_receive_skb, XDP-action result, or properly-discarded; no silent drops.

## Hardening

(Inherits row-1 features from `drivers/net/ethernet/00-overview.md` § Hardening.)

mlx5-en specific reinforcement:

- **Per-channel NAPI poll-budget** — bounded; cross-ref `napi_budget.tla`.
- **Per-channel page-pool DMA-mask validated** — defense against silent swiotlb-bounce.
- **aRFS per-flow filter count cap** — per-NIC max-rfs-flow capped by ASIC TCAM; over-cap returns -ENOSPC.
- **Firmware command rate-limited** — per-NIC firmware cmd-q + completion timeout (default 60s); defense against firmware hang.
- **TLS-TX/RX key handling HW-only** — keys passed via firmware command; never visible to userspace post-install; HW state-fetch sealed.
- **IPSec offload key handling** same.
- **devlink reload-fw + flash CAP_NET_ADMIN + CAP_SYS_RAWIO + LSM mediation** — defense against malicious firmware persistence.
- **Per-VF FLR (Function Level Reset)** isolates faulty VF without affecting siblings.
- **Per-channel XDP-SQ wraparound bounded** — defense against XDP-TX-flood causing DoS.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — mlx5e Rx WQE fragment slabs (mlx5e_rq_frag_info, mlx5e_mpw_info) and DEVLINK param blobs flagged whitelist-only; skb headroom sized so usercopy bound-checks the actual XDP_PACKET_HEADROOM + NET_IP_ALIGN.
- **PAX_KERNEXEC** — mlx5e NAPI poll, mlx5e_handle_rx_cqe, mlx5e_xmit, and the mlx5_core EQ ISR run W^X; no FW-loaded blob is mapped writable+executable into kernel text.
- **PAX_RANDKSTACK** — per-MSI-X-completion entry randomizes kernel stack so a hostile peer cannot pivot ROP off a stable offset in the mlx5e Rx path.
- **PAX_REFCOUNT** — mlx5e_priv, mlx5_core_dev, page_pool refs all use saturating refcount_t; defense against refcount overflow from devlink reload storm or RDMA QP teardown race.
- **PAX_MEMORY_SANITIZE** — Rx fragment / WQE / CQE-pool slabs poison-on-free; defense against per-VF re-use of a freed fragment leaking peer-VF traffic.
- **PAX_UDEREF** — mlx5 netlink, devlink, and ethtool ioctl entry points enforce SMAP-equivalent split address spaces on copy_from/to_user of mlx5_ifc_* layout structures.
- **PAX_RAP/kCFI** — mlx5e_profile->init/cleanup/update_rx, mlx5e_rx_handlers, mlx5_core cmd_ops vtables CFI-protected; defense against forged function pointer in profile struct after FW reset.
- **GRKERNSEC_HIDESYM** — mlx5_ifc_* layout symbols and mlx5e_* helper addresses hidden from /proc/kallsyms to unprivileged readers.
- **GRKERNSEC_DMESG** — dmesg restricted; mlx5e link-event, FW-health, and "FW error syndrome" prints do not leak HCA layout to non-CAP_SYSLOG readers.
- **mlx5_core CAP_NET_ADMIN** — mlx5e netdev configuration (ethtool, devlink port, ring resize, channel count) gated on CAP_NET_ADMIN on the netdev's netns.
- **MLX5 firmware-blob signature** — FW image (mlxfw / fw_reset) verified against Mellanox signing key before MFRL/MIRC load; defense against per-rogue-firmware persistent rootkit in HCA.
- **DEVLINK PAX_USERCOPY** — devlink param get/set / health-reporter dump buffers usercopy-whitelisted so an oversized devlink reply cannot leak adjacent slab.
- **Per-flow-steering rule CAP_NET_ADMIN** — TC offload / ct-offload / mirroring rule install gated; defense against per-unprivileged-tap of foreign VF traffic.
- **Per-SR-IOV VF spawn CAP_SYS_ADMIN on PF netns** — defense against unprivileged enabling of VFs against a PF assigned to a different netns.
- Rationale: mlx5_core/mlx5e expose a large devlink + ethtool + RDMA + flow-steering UAPI fed by a vendor-signed firmware blob; grsec posture combines CAP_NET_ADMIN/CAP_SYS_ADMIN gates with FW-signature enforcement, CFI on the profile/handler vtables, and usercopy-whitelisting on devlink dumps.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mlx5_core foundation (covered in `drivers/net/ethernet/mellanox-mlx5-core.md` future Tier-3)
- mlx5 TC offload (covered in `mellanox-mlx5-en-tc.md` future Tier-3)
- mlx5 SR-IOV switchdev representors (covered in `mellanox-mlx5-en-rep.md` future Tier-3)
- mlx5 eswitch (covered in `mellanox-mlx5-eswitch.md` future Tier-3)
- mlx5 flow steering core (covered in `mellanox-mlx5-fs.md` future Tier-3)
- mlx5 RDMA path (covered in `drivers/infiniband/hw-mlx5.md` future Tier-3)
- 32-bit-only paths
- Implementation code
