---
title: "Tier-2: drivers/net/ethernet — Ethernet driver vendor-LLD wrapper (Intel + Mellanox + Broadcom + Marvell + Realtek + Aquantia + virtio-net + …)"
tags: ["tier-2", "drivers", "net", "ethernet", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the Ethernet vendor-LLD wrapper — every Ethernet NIC driver in tree consumes the netdev framework + ethtool + napi + xdp / xdp-mb + tc / tc-mqprio / tc-flower offload + devlink + page-pool + MSI-X + AF_XDP zerocopy + xdp-redirect + bpf-offload + dim (dynamic interrupt moderation). This Tier-2 wraps the LLD layer; the upper-layer netdev framework lives in `net/00-overview.md` (the protocol stack) and `drivers/net/Makefile`-rooted infrastructure (page-pool, dim, libeth, libie, bonding, bridge, tun, veth, vxlan) is partly in `drivers/net/` peers.

The vendor-LLD set is huge (~80 vendors, hundreds of drivers); v0 keeps a maintenance core covering the dominant deployments + hyperscaler-friendly gear:

- **Intel** (`intel/`): `e100.c` (legacy 100Mb), `e1000` (legacy 1Gb PCI), `e1000e/` (PCIe 1Gb consumer/embedded), `igb/` (server 1Gb 82576/82580/I210/I211/I350), `igc/` (2.5Gb I225/I226), `ixgbe/` + `ixgbevf/` (10/25Gb 82598/82599/X540/X550), `i40e/` + `iavf/` (10/25/40Gb XL710/X710/XXV710/X722), `ice/` (10/25/50/100Gb E810 Columbiaville), `idpf/` (Intel Data-Plane Function — Mount Evans / IPU), `fm10k/` (FM10000 datacenter switch), shared libs `libeth/` + `libie/`
- **Mellanox / NVIDIA** (`mellanox/`): `mlx4/` (legacy ConnectX-3), `mlx5/` (ConnectX-4/5/6/7/8 + BlueField DPU host stack — the biggest single driver in the tree), `mlxsw/` (Spectrum switch ASIC), `mlxbf_gige/` (BlueField OOB management), `mlxfw/` (firmware update lib)
- **Broadcom** (`broadcom/`): `bnxt/` (NetXtreme-C/E 10/25/50/100/200Gb), `tg3.c` (legacy NetXtreme), `bgmac*` (SoC), `b44.c` (legacy), `genet/` (GENET SoC GbE), `bnx2x/` (legacy 10Gb), `bnx2.c` (legacy), `bcmsysport.c`, `asp2/` (Asp2 SoC)
- **Marvell** (`marvell/`): `mvneta.c`, `mvpp2/` (Marvell Armada Ethernet), `mvmdio.c`, `octeontx2/` (OcteonTX2 datacenter), `octeon_ep/` (Octeon endpoint), `prestera/` (Prestera switch ASIC), `pxa168_eth.c`, `skge.c`, `sky2.c`
- **Realtek** (`realtek/`): `r8169_main.c` + variant files (RTL8168/RTL8169/RTL8125 — the dominant consumer/SOHO 1/2.5GbE)
- **Aquantia** (`aquantia/atlantic/`): AQC107/AQC108/AQC112 (consumer 5/10GbE)
- **Amazon** (`amazon/ena/`): ENA — AWS Elastic Network Adapter
- **Google** (`google/gve/`): gVNIC — Google VM NIC
- **Microsoft** (`microsoft/mana/` — under `drivers/net/ethernet/microsoft/`): MANA — Azure Network Adapter
- **Hyper-V** (`netvsc.c` — actually under `drivers/net/hyperv/`): hyperv-vsc-netdev (Hyper-V synthetic NIC)
- **Cisco** (`cisco/enic/`): UCS VIC
- **Chelsio** (`chelsio/`): T4/T5/T6 (cxgb4 + iwarp + iscsi + offload TLS)
- **Emulex / Brocade** (`emulex/benet/`): be2net (legacy)
- **HiSilicon** (`hisilicon/hns3/`): HiSilicon HNS3 (datacenter)
- **Huawei** (`huawei/hinic/`)
- **IBM** (`ibm/ibmvnic.c` + `ibm/ibmveth.c`): pSeries virtual NICs (out of v0 — PowerVM)
- **Pensando / AMD** (`pensando/ionic/`): IONIC — AMD Pensando DSC
- **AMD-Xilinx** (`xilinx/` + `amd/`): Xilinx AXI Ethernet, AMD/Xilinx Versal
- **Fungible** (`fungible/funeth/`): Fungible DPU (since merged-out — keep status check)
- **virtio-net** (`drivers/net/virtio_net.c`): virtio NIC (the dominant guest NIC — every cloud VM ships this)
- **Compile-gated-off legacy / out-of-v0**: 3com (`3com/`), 8390 (`8390/` — NE2000 era), AMD legacy `amd/`, agere, aeroflex/grlib, alteon, altera, arc, calxeda, cirrus, cortina, davicom, dec, dlink, faraday, fealnx.c, freescale (PowerPC SoC), hamradio (`drivers/net/hamradio/`), i825xx (Intel 825xx era), korina (MIPS SoC), litex, micrel, microchip, moxa, mscc, myricom, neterion (Exar), netronome (Agilio — keep until upstream lands removal), ni, nvidia (forcedeth), nxp (LS-series PowerPC SoC), oki-semi, pasemi (PA-Semi), pensando (keep — IONIC), qlogic legacy `netxen_nic` and `qed/qede` (keep), rocker, sfc (Solarflare — keep), silan, sis, smsc (legacy), socionext, stmicro, sun (cassini, sungem, sunhme, sunqe, sunvnet — out of v0), synopsys, tehuti, ti (cpsw / am65 / icssg — keep for embedded), toshiba, tundra, vertexcom, via (rhine — keep until removal lands upstream), wangxun (txgbe — keep), wiznet, xircom, xscale.

Heavily cross-referenced from `net/00-overview.md` (every netdev consumer there), `kernel/bpf/00-overview.md` (xdp + bpf-offload), `drivers/pci/00-overview.md` (PCIe binding + SR-IOV PF/VF split), `drivers/iommu/00-overview.md` + `drivers/vfio/00-overview.md` (passthrough), `drivers/dma-buf/` (af_xdp memory provider), `kernel/dma/00-overview.md` (page-pool + dma_map_page), `kernel/irq/00-overview.md` (msi-domain alloc one IRQ per RX/TX queue).

### Out of Scope

- Per-Tier-3 (Phase D)
- ARM-only SoC NICs (`freescale/`, `apm/`, `cadence/`, `cavium/`, `cirrus/`, `cortina/`, `ezchip/`, `faraday/`, `fungible/`, `hisilicon/hisi_*`, `nxp/`, `oki-semi/`, `pasemi/`, `qualcomm/`, `renesas/`, `rocker/`, `seeq/`, `silan/`, `sis/`, `socionext/`, `sun/`, `synopsys/`, `tehuti/`, `toshiba/`, `tundra/`, `vertexcom/`, `wiznet/`, `xircom/`, `xscale/`, etc.) — out of v0
- Wireless drivers (`drivers/net/wireless/`) — separate Tier-2 wrapper future
- Bluetooth (`drivers/bluetooth/`) — separate Tier-2 wrapper future
- WAN drivers (PPP/SLIP/etc., `drivers/net/wan/`) — separate Tier-2 wrapper future
- Hamradio (`drivers/net/hamradio/`) — out of v0
- IBM PowerVM virtual NICs (`ibm/ibmvnic.c`, `ibm/ibmveth.c`) — out of v0
- 32-bit-only paths
- Implementation code

### components

### Per-vendor drivers (active maintenance set in v0)

(See Summary list above for the core vendor list. Each vendor sub-driver has its own per-vendor Tier-3 deferred to Phase D — per-vendor Tier-3 docs are large and per-driver; this Tier-2 enumerates them.)

### Shared infrastructure consumed by vendor LLDs

- **page-pool** (`net/core/page_pool.c` — lives outside `drivers/net/` but used by every modern vendor LLD): per-NAPI page recycling pool with optional DMA pre-mapping; backbone of zero-alloc RX path and AF_XDP/zero-copy
- **DIM** (`lib/dim/` — Dynamic Interrupt Moderation library): adaptive RX/TX coalescing controller used by mlx5, ice, ena, gve, others
- **libeth** (`drivers/net/ethernet/intel/libeth/`): Intel-shared lib (queue setup, RSS context, etc.)
- **libie** (`drivers/net/ethernet/intel/libie/`): Intel-shared lib for newer drivers (idpf, ice in some paths)
- **mlxfw** (`drivers/net/ethernet/mellanox/mlxfw/`): Mellanox firmware-update lib
- **netdev_features** + **ethtool** + **dim** + **devlink** + **napi** + **xdp** + **page_pool**: framework consumers — see `net/00-overview.md`

### scope

This Tier-2 governs:
- `/home/doll/linux-src/drivers/net/ethernet/` (~80 vendor subdirs, hundreds of drivers)
- `/home/doll/linux-src/drivers/net/virtio_net.c` (the single dominant guest NIC driver)
- `/home/doll/linux-src/drivers/net/hyperv/netvsc*.c` (Hyper-V synthetic NIC — physically under drivers/net/hyperv but conceptually part of "ethernet vendor LLDs")
- Per-LLD private headers (kept per-driver subdir)

Public framework headers (`include/linux/netdevice.h`, `include/linux/etherdevice.h`, `include/linux/ethtool.h`, `include/net/page_pool/*.h`, `include/net/devlink.h`, `include/net/xdp.h`, `include/net/xdp_sock.h`, `include/linux/bpf.h`) are governed by `net/00-overview.md` — this Tier-2 only consumes them.

LLD-shared infra under `drivers/net/` peers (bonding, team, dummy, tun, veth, ifb, vrf, geneve, vxlan, gre, macvlan, macvtap, ipvlan, nlmon, ppp, pptp, slip, wireguard, xen-netfront/xen-netback, virtio_net) is at the same physical level but conceptually each is its own Tier-3 future under a future `drivers/net/00-overview.md` Tier-2 wrapper or directly under `net/`.

### compatibility contract — outline

### `MODULE_DEVICE_TABLE(pci, ...)` per-LLD

Each PCI-based LLD's match-table byte-identical so depmod/modprobe load the right driver from the right modalias.

### ethtool ioctl + netlink wire format

Per-LLD ethtool callbacks (`get_drvinfo`, `get_link_ksettings`, `get_ringparam`, `set_ringparam`, `get_coalesce`, `set_coalesce`, `get_pauseparam`, `set_pauseparam`, `get_sset_count`, `get_strings`, `get_ethtool_stats`, `get_module_info`, `get_module_eeprom`, `get_dump_flag`, `get_dump_data`, `set_dump`, `get_priv_flags`, `set_priv_flags`, `flash_device`, `reset`, `get_eee`, `set_eee`, `nway_reset`, `get_link`, `get_pauseparam_ex`, `get_per_queue_coalesce`, `set_per_queue_coalesce`, `get_phy_tunable`, `set_phy_tunable`, `get_tunable`, `set_tunable`, `get_rxfh_indir_size`, `get_rxfh_key_size`, `get_rxfh`, `set_rxfh`, `get_rxnfc`, `set_rxnfc`, `get_channels`, `set_channels`, `self_test`, `get_link_ext_state`, `get_eth_phy_stats`, `get_eth_mac_stats`, `get_eth_ctrl_stats`, `get_rmon_stats`, `get_module_power_mode`, `set_module_power_mode`, `get_fec_stats`, `get_fecparam`, `set_fecparam`, `get_link_ext_stats`, `get_mm`, `set_mm`, `get_mm_stats`, `get_ts_info`, `get_ts_stats`, ...) byte-identical so ethtool userspace tool + Cumulus Linux switchd + Sonic + datacenter monitoring tools work unchanged.

### sysfs surface

Per-netdev `/sys/class/net/<dev>/`:
- ID/state: `address`, `addr_assign_type`, `addr_len`, `broadcast`, `carrier`, `carrier_changes`, `carrier_down_count`, `carrier_up_count`, `dev_id`, `dev_port`, `dormant`, `flags`, `gro_flush_timeout`, `gso_ipv4_max_size`, `gso_max_segs`, `gso_max_size`, `ifalias`, `ifindex`, `iflink`, `link_mode`, `mtu`, `name_assign_type`, `napi_defer_hard_irqs`, `netdev_group`, `operstate`, `phys_port_id`, `phys_port_name`, `phys_switch_id`, `proto_down`, `speed`, `tsq_max`, `tx_queue_len`, `type`, `uevent`
- `device/` symlink to PCI/USB parent
- `queues/{rx-N,tx-N}/{rps_cpus,rps_flow_cnt,xps_cpus,xps_rxqs,byte_queue_limits/,traffic_class,tx_maxrate,...}`
- `statistics/{rx_bytes,rx_packets,rx_dropped,rx_errors,rx_compressed,rx_crc_errors,rx_fifo_errors,rx_frame_errors,rx_length_errors,rx_missed_errors,rx_over_errors,multicast,collisions,tx_bytes,tx_packets,tx_dropped,tx_errors,tx_compressed,tx_aborted_errors,tx_carrier_errors,tx_fifo_errors,tx_heartbeat_errors,tx_window_errors}`

Layout + content byte-identical so Prometheus node-exporter, ifconfig, ip-link, sosreport consume unchanged.

### `/proc/net/dev`

Legacy procfs counters; format byte-identical (still consumed by older monitoring scripts).

### Per-vendor devlink

`devlink dev show pci/0000:01:00.0` + `devlink port show` + `devlink resource show` + `devlink param show` byte-identical for every vendor LLD that registers devlink. Used heavily by ice/mlx5/ena/bnxt for runtime config + reload + flash.

### XDP-DRV mode

Each modern LLD MUST support XDP-DRV (native-driver XDP) — `BPF_XDP_FLAGS_HW_MODE` rejected unless driver supports HW-offload (mlx5+nfp). DRV vs SKB mode selection identical.

### AF_XDP zerocopy

Vendor LLDs supporting AF_XDP zerocopy: mlx5, ice, ixgbe, i40e, ena, igc, stmmac. UMEM registration + Tx descriptor + Rx descriptor format byte-identical so libxdp + xdp-tools + DPDK work unchanged.

### tc / tc-flower offload

Per-LLD ndo_setup_tc + flow_block_offload callbacks. Vendor LLDs supporting flower offload: mlx5, ice, nfp, bnxt, sfc. tc-flower → HW table programming wire format on the kernel side identical so iproute2 `tc filter add dev <dev> ingress flower ... action mirred ... skip_sw` works unchanged.

### TLS offload

Per-LLD TLS-RX + TLS-TX offload (mlx5, nfp, bnxt). `setsockopt(SOL_TLS, TLS_TX/RX, ...)` returns offloaded handle when supported. Cross-ref `net/tls/`.

### IPsec offload

Per-LLD IPsec inline + crypto offload (mlx5, ice). `xfrm_state_add` with `XFRMA_OFFLOAD_DEV` returns offloaded SA when supported. Cross-ref `net/xfrm/`.

### PTP hardware timestamping

Per-LLD PHC clock registered via ptp_clock_register (mlx5, ice, ixgbe, igb, igc, i40e, bnxt, ena, mvpp2, etc.). `/dev/ptp<N>` chardev appears; phc2sys synchronizes. Cross-ref `kernel/time/`.

### RoCE / iWARP integration

Per-LLD RDMA-aux-bus split (mlx5, bnxt, ice → en+rdma sub-drivers via auxiliary bus from `drivers/base/auxiliary.md`). Cross-ref future `drivers/infiniband/00-overview.md`.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally; per-LLD Tier-3 docs are large and per-driver. This list is the v0 maintenance plan; non-listed legacy LLDs stay compile-gated-off.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/net/ethernet/intel-libeth.md` | `intel/libeth/`: shared Intel queue/RSS lib |
| `drivers/net/ethernet/intel-libie.md` | `intel/libie/`: shared Intel newer-gen lib |
| `drivers/net/ethernet/intel-e1000e.md` | `intel/e1000e/`: PCIe 1Gb consumer/embedded |
| `drivers/net/ethernet/intel-igb.md` | `intel/igb/`: server 1Gb 82576-I350 |
| `drivers/net/ethernet/intel-igbvf.md` | `intel/igbvf/`: igb VF |
| `drivers/net/ethernet/intel-igc.md` | `intel/igc/`: 2.5Gb I225/I226 |
| `drivers/net/ethernet/intel-ixgbe.md` | `intel/ixgbe/`: 10/25Gb 82599/X540/X550 |
| `drivers/net/ethernet/intel-ixgbevf.md` | `intel/ixgbevf/`: ixgbe VF |
| `drivers/net/ethernet/intel-i40e.md` | `intel/i40e/`: 10/25/40Gb X710/XL710/X722 |
| `drivers/net/ethernet/intel-iavf.md` | `intel/iavf/`: i40e VF |
| `drivers/net/ethernet/intel-ice.md` | `intel/ice/`: 10/25/50/100Gb E810 |
| `drivers/net/ethernet/intel-idpf.md` | `intel/idpf/`: Mount Evans IPU |
| `drivers/net/ethernet/intel-fm10k.md` | `intel/fm10k/`: FM10000 datacenter switch |
| `drivers/net/ethernet/mellanox-mlx4.md` | `mellanox/mlx4/`: ConnectX-3 (legacy) |
| `drivers/net/ethernet/mellanox-mlx5-core.md` | `mellanox/mlx5/core/`: mlx5 core (foundation) |
| `drivers/net/ethernet/mellanox-mlx5-en.md` | `mellanox/mlx5/core/en*`: en sub-driver — netdev path |
| `drivers/net/ethernet/mellanox-mlx5-en-tc.md` | mlx5 en/tc/ subdirs: TC offload + flower + ct |
| `drivers/net/ethernet/mellanox-mlx5-en-rep.md` | mlx5 en_rep: SR-IOV switchdev representors |
| `drivers/net/ethernet/mellanox-mlx5-fpga.md` | mlx5 FPGA + IPSec offload (legacy ConnectX-5) |
| `drivers/net/ethernet/mellanox-mlx5-eswitch.md` | mlx5 eswitch + offloaded modes |
| `drivers/net/ethernet/mellanox-mlx5-fs.md` | mlx5 flow steering core |
| `drivers/net/ethernet/mellanox-mlxsw.md` | `mellanox/mlxsw/`: Spectrum switch ASIC |
| `drivers/net/ethernet/mellanox-mlxbf-gige.md` | `mellanox/mlxbf_gige/`: BlueField OOB |
| `drivers/net/ethernet/mellanox-mlxfw.md` | `mellanox/mlxfw/`: firmware update |
| `drivers/net/ethernet/broadcom-bnxt.md` | `broadcom/bnxt/`: NetXtreme-C/E |
| `drivers/net/ethernet/broadcom-tg3.md` | `broadcom/tg3.c`: NetXtreme legacy |
| `drivers/net/ethernet/broadcom-genet.md` | `broadcom/genet/`: SoC GbE |
| `drivers/net/ethernet/broadcom-bnx2x.md` | `broadcom/bnx2x/`: 10Gb legacy |
| `drivers/net/ethernet/marvell-mvneta.md` | `marvell/mvneta.c`: Armada Eth |
| `drivers/net/ethernet/marvell-mvpp2.md` | `marvell/mvpp2/`: Armada Eth v2 |
| `drivers/net/ethernet/marvell-octeontx2.md` | `marvell/octeontx2/`: OcteonTX2 |
| `drivers/net/ethernet/marvell-prestera.md` | `marvell/prestera/`: Prestera switch |
| `drivers/net/ethernet/realtek-r8169.md` | `realtek/r8169_main.c` + variants |
| `drivers/net/ethernet/aquantia-atlantic.md` | `aquantia/atlantic/`: AQC107/108/112 |
| `drivers/net/ethernet/amazon-ena.md` | `amazon/ena/`: AWS ENA |
| `drivers/net/ethernet/google-gve.md` | `google/gve/`: GCP gVNIC |
| `drivers/net/ethernet/microsoft-mana.md` | `microsoft/mana/`: Azure MANA |
| `drivers/net/ethernet/cisco-enic.md` | `cisco/enic/`: UCS VIC |
| `drivers/net/ethernet/chelsio-cxgb4.md` | `chelsio/cxgb4/`: T4/T5/T6 |
| `drivers/net/ethernet/hisilicon-hns3.md` | `hisilicon/hns3/`: HNS3 datacenter |
| `drivers/net/ethernet/huawei-hinic.md` | `huawei/hinic/` |
| `drivers/net/ethernet/pensando-ionic.md` | `pensando/ionic/`: AMD Pensando DSC |
| `drivers/net/ethernet/sfc.md` | `sfc/`: Solarflare/AMD-Xilinx |
| `drivers/net/ethernet/ti-cpsw-am65.md` | `ti/`: AM65/J7/AM64 CPSW + ICSSG |
| `drivers/net/ethernet/wangxun-txgbe.md` | `wangxun/txgbe/`: WangXun 10Gb |
| `drivers/net/ethernet/qlogic-qede.md` | `qlogic/qede/`: QED/QEDE |
| `drivers/net/ethernet/virtio-net.md` | `drivers/net/virtio_net.c`: virtio guest NIC |
| `drivers/net/ethernet/hyperv-netvsc.md` | `drivers/net/hyperv/netvsc*.c`: Hyper-V synthetic NIC |

### compatibility outline (top-level)

- REQ-O1: Per-LLD `MODULE_DEVICE_TABLE(pci, ...)` byte-identical so depmod resolves correctly.
- REQ-O2: Per-LLD `net_device_ops` + `ethtool_ops` + `xdp_ops` + `dcbnl_ops` + `tlsdev_ops` + `xfrmdev_ops` callbacks source-compatible (in-tree LLDs build unchanged).
- REQ-O3: `/sys/class/net/<dev>/` per-netdev sysfs surface byte-identical; `/proc/net/dev` legacy procfs byte-identical.
- REQ-O4: ethtool ioctl + netlink wire format byte-identical so `ethtool` userspace tool consumes unchanged on every LLD.
- REQ-O5: devlink wire format byte-identical for every LLD that registers devlink (devlink-cli + Cumulus switchd consume unchanged).
- REQ-O6: XDP-DRV native + AF_XDP zerocopy work identically on the listed LLDs supporting them (mlx5, ice, ixgbe, i40e, ena, igc, virtio-net).
- REQ-O7: tc / tc-flower offload (mlx5, ice, nfp, bnxt, sfc) wire-identical so `tc filter add ... skip_sw` works unchanged.
- REQ-O8: TLS-TX/RX offload (mlx5, nfp, bnxt) and IPsec offload (mlx5, ice) work identically.
- REQ-O9: PTP hardware timestamping (mlx5, ice, ixgbe, igb, igc, i40e, bnxt, ena, …) via ptp_clock_register + `/dev/ptp<N>` works unchanged.
- REQ-O10: RDMA-aux-bus split (mlx5, bnxt, ice) registers en + rdma sub-drivers correctly via auxiliary bus.
- REQ-O11: TLA+ models declared at this Tier-2 (per-LLD ring management + NAPI poll-budget + page-pool refcount + AF_XDP UMEM completion-ring acq-rel ordering).
- REQ-O12: Verus/Creusot Layer-4 functional contracts on per-LLD descriptor-ring head/tail invariants + DMA-mask coherence.
- REQ-O13: Hardening: row-1 features applied per `00-security-principles.md`, with NIC-LLD-specific reinforcement (per-vendor firmware-flash gated CAP_NET_ADMIN + LSM, devlink reload mediated, ethtool flash_device gated).

### acceptance criteria (top-level)

- [ ] AC-O1: `ethtool -i ens0f0` (Intel ice or mlx5) returns byte-identical strings to upstream baseline. (covers REQ-O4)
- [ ] AC-O2: `ip link show` enumerates every NIC matching upstream device naming + sysfs paths. (covers REQ-O3)
- [ ] AC-O3: Stress test: `iperf3` over mlx5 ConnectX-6 100GbE achieves line-rate. (covers REQ-O2)
- [ ] AC-O4: XDP-DRV test: `xdp-loader load -m native ens0f0 xdp_redirect.o` works on mlx5 + ice + ixgbe + i40e + virtio-net. (covers REQ-O6)
- [ ] AC-O5: AF_XDP zerocopy test: `xdpsock --rx --tx -i ens0f0 --zero-copy` achieves pps within 5% of upstream baseline. (covers REQ-O6)
- [ ] AC-O6: tc-flower offload test: `tc filter add dev ens0f0 ingress flower src_mac aa:bb:cc:dd:ee:ff action drop skip_sw` succeeds on mlx5 + ice. (covers REQ-O7)
- [ ] AC-O7: TLS-RX offload test: `setsockopt(SOL_TLS, TLS_RX, &cipher, sizeof(cipher))` followed by data RX → CPU bypass on mlx5 + nfp. (covers REQ-O8)
- [ ] AC-O8: PTP test: `phc2sys -s ens0f0 -O 0 -m` synchronizes system clock to PHC; offset converges to < 1µs. (covers REQ-O9)
- [ ] AC-O9: SR-IOV test: `echo 4 > /sys/bus/pci/devices/.../sriov_numvfs` on ice + iavf binds 4 VFs that enumerate + carry traffic. (covers REQ-O2)
- [ ] AC-O10: virtio-net guest in qemu boots + connects to bridge, receives DHCP, throughput within 2% upstream. (covers REQ-O2)
- [ ] AC-O11: kselftest `tools/testing/selftests/net/forwarding/*` passes on mlxsw or ice. (covers REQ-O11, REQ-O12)
- [ ] AC-O12: kselftest `tools/testing/selftests/drivers/net/*` passes (per-driver basic tests). (covers REQ-O11, REQ-O12)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/net-eth/page_pool.tla` | `drivers/net/ethernet/intel-libeth.md` (proves: per-NAPI page-pool refcount + recycle path; concurrent skb-free returning page to pool + NAPI-poll allocating from pool never produces double-recycle or leaked page) |
| `models/net-eth/napi_budget.tla` | `drivers/net/ethernet/intel-ice.md` (proves: NAPI poll-budget enforcement under sustained burst — poll → budget exceeded → reschedule via softirq → poll resumes; no missed completion / no infinite-loop in poll) |
| `models/net-eth/af_xdp_umem.tla` | `drivers/net/ethernet/mellanox-mlx5-en.md` + `drivers/net/ethernet/intel-ice.md` (proves: AF_XDP UMEM fill-ring + completion-ring + tx-ring + rx-ring acq-rel ordering between userspace producer/consumer and kernel NAPI; no descriptor lost or double-used) |
| `models/net-eth/sriov_pf_vf.tla` | `drivers/net/ethernet/intel-iavf.md` + `drivers/net/ethernet/mellanox-mlx5-eswitch.md` (proves: PF↔VF mailbox + per-VF state machine; concurrent PF reset + VF probe + VF unbind never produces partial state or stale VF bound to dead PF) |
| `models/net-eth/devlink_reload.tla` | `drivers/net/ethernet/intel-ice.md` (proves: devlink-reload-fw_activate state machine — ports detached, fw activated, ports reattached; no in-flight skb leaked; failure-path rollback complete) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/net/ethernet/intel-libeth.md` | RX descriptor-ring head/tail invariant: `next_to_clean` <= `next_to_use`, both modulo ring-size; descriptor at `next_to_clean` is owned by HW until DD bit set |
| `drivers/net/ethernet/mellanox-mlx5-core.md` | mlx5 command-queue: each command has unique slot + completion observed exactly once; concurrent submit from N CPUs serialized via cmdif lock |
| `drivers/net/ethernet/virtio-net.md` | virtio-net virtqueue invariant: descriptor-chain head + length match the underlying skb; available + used rings respect virtio spec (memory-barrier between produce-index update + value write) |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-netdev + per-page-pool + per-vf-rep + per-aux-device refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-LLD `net_device_ops`, `ethtool_ops`, `xdp_ops`, `dcbnl_ops`, `tlsdev_ops`, `xfrmdev_ops` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-ring head/tail arithmetic + skb-frag size + page-pool slot count use checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-LLD ops vtables + per-LLD HW-cmd dispatch tables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed netdev priv data cleared (carries crypto keys for TLS offload, IPsec SA pointers) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | NIC LLDs have no JIT — N/A | § Mandatory |

### Row-2 (LSM-stackable) features for NIC LLDs

LSM hooks called: file-LSM hooks on `/sys/class/net/.../`, `/sys/bus/pci/devices/.../sriov_numvfs`; per-syscall netlink LSM hooks (already covered in `net/00-overview.md`).

GR-RBAC adds:
- Per-role disallow ethtool `flash_device` (firmware update).
- Per-role disallow devlink reload-fw.
- Per-role disallow `sriov_numvfs` write on PCI parent.
- Per-role audit of every TC-offload flow installation (NIC could leak forwarded traffic — auditing flow setup catches misconfig).
- Per-role disallow per-LLD vendor-specific debugfs writes.

### NIC-LLD-specific reinforcement

- **Firmware-flash gated**: `ethtool flash_device` + devlink fw_activate + per-vendor `mlxlink`/`ice-tools` commands require CAP_NET_ADMIN + CAP_SYS_RAWIO + LSM mediation. Defense against malicious firmware persistence.
- **devlink param `running` vs `runtime`**: devlink param changes that affect firmware behavior on next reset visible at `running` value; current at `runtime`. Rookery enforces `running`-effect changes go through reload + audit.
- **VF token / PCI VF binding**: `vfio-pci` binding of a VF requires owning the parent PF via `VFIO_DEVICE_FEATURE_PCI_VF_TOKEN` (defense against unauthorized passthrough of someone else's VF).
- **Per-LLD page-pool DMA mask**: page-pool init validates dma_mask coverage; refuse pool create when dma_mask insufficient (defense against silent swiotlb-bounce thrashing).
- **AF_XDP UMEM permission**: `bind(XDP_USE_NEED_WAKEUP)` requires CAP_NET_RAW + per-iface allowlist (defense against userspace stealing all RX from an interface).
- **TC-flower offload program-counter cap**: per-NIC max-offloaded-flow-count enforced (default 128k for mlx5, 16k for ice) so flow-DOS doesn't exhaust HW TCAM.
- **TLS-RX offload key handling**: TLS keys passed to NIC HW are HW-only after install (kernel zeroes after copy-out); HW state-fetch for migration uses sealed channel.
- **SR-IOV trust bit** (`ip link set <pf> vf <n> trust on`) gates whether VF can program promiscuous mode / MAC change / VLAN. Default OFF; CAP_NET_ADMIN to enable.

