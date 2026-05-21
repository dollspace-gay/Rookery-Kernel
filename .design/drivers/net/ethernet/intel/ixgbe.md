# Tier-3: drivers/net/ethernet/intel/ixgbe/{ixgbe_main,ixgbe_common,ixgbe_82598,ixgbe_82599,ixgbe_x540,ixgbe_x550,ixgbe_e610,ixgbe_lib,ixgbe_ethtool,ixgbe_ipsec,ixgbe_fcoe,ixgbe_sriov,ixgbe_mbx,ixgbe_phy,ixgbe_ptp,ixgbe_dcb,ixgbe_dcb_82598,ixgbe_dcb_82599,ixgbe_dcb_nl,ixgbe_xsk,ixgbe_fw_update,ixgbe_sysfs,ixgbe_debugfs}.c + devlink/ — Intel 82598/82599/X540/X550/X550EM/E610 10/25 Gbps NICs

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
upstream-paths:
  - drivers/net/ethernet/intel/ixgbe/ixgbe_main.c                (~16000 lines: netdev ops, NAPI, RX/TX rings, IRQ, watchdog, suspend/resume)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_common.c              (~5500 lines: cross-silicon helpers — EEPROM, MAC ops, link mgmt)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_82598.c               (~2000 lines: 82598EB 10G silicon — first-gen, oldest still supported)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_82599.c               (~3000 lines: 82599 — 10G, most widely deployed historically)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_x540.c                (X540 — 10GBase-T)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_x550.c                (~3000 lines: X550 / X552/X553 — 10G + 2.5G + 1G)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_e610.c                (~5500 lines: E610 — newest 10/25G with Admin-Q FW interface)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_lib.c                 (cross-version helpers)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_ethtool.c             (~3500 lines: ethtool ops — stats, RSS, coalesce, link, fec)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_ipsec.c               (IPsec inline crypto offload)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_fcoe.c                (FCoE offload)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_sriov.c               (SR-IOV PF→VF management; sibling `ixgbevf` is the VF driver)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_mbx.c                 (PF↔VF mailbox)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_phy.c                 (PHY mgmt — Intel + 3rd-party PHYs)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_ptp.c                 (PHC)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_dcb*.c                (DCB — PFC/ETS for lossless FCoE/RoCE)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_xsk.c                 (AF_XDP zerocopy)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_fw_update.c           (E610 FW update via Admin-Q)
  - drivers/net/ethernet/intel/ixgbe/ixgbe_sysfs.c, ixgbe_debugfs.c
  - drivers/net/ethernet/intel/ixgbe/devlink/devlink.c           (devlink integration)
  - drivers/net/ethernet/intel/ixgbe/devlink/region.c            (devlink region snapshot)
-->

## Summary

ixgbe is the Intel 10/25 Gigabit Ethernet driver — covers the entire Intel 10G NIC family from 82598EB (2008, the first 10G silicon shipped at scale) through 82599 (the most-deployed historically), X540 (10GBase-T), X550/X552/X553, and E610 (newest 10/25G with modern Admin-Q FW interface). Deployed across decades of datacenter, financial trading, and enterprise networking — still ships today on many server motherboards as the default network silicon. While newer 100G silicon is Mellanox/Broadcom/Marvell, the installed base of ixgbe-driven 10G is huge: every cloud-mod server pre-2018, most non-cloud enterprise deployments, all the OEM management interfaces (iLO/iDRAC/BMC out-of-band networks).

ixgbe supports:
- **PF (Physical Function)** — primary host-side driver. Up to 4 ports per PCIe device.
- **SR-IOV** — up to 64 VFs per port; per-VF mailbox; sibling `ixgbevf` driver runs in VFs.
- **DCB / FCoE** — Data Center Bridging (PFC + ETS) for lossless FCoE. Has its own offload engine.
- **IPsec inline** — AES-GCM AH/ESP offload (newer silicon).
- **PTP** — hardware timestamping.
- **AF_XDP** — zerocopy XDP socket.
- **FW update via Admin-Q** — newer E610 silicon uses an admin queue protocol (ICE-derived) for FW updates.

This Tier-3 covers ~48000 lines / ~25 .c files.

## Rust translation posture

ixgbe is the older sibling of `ice` (Intel 100G driver) and shares some patterns (Intel-style HAL abstraction via per-silicon `ixgbe_hw_ops`). Translation:

- **`ixgbe-core` crate** — netdev, NAPI, RX/TX rings, IRQ, link mgmt, ethtool, SR-IOV.
- **Per-silicon trait `IxgbeSilicon`** — implemented by `ixgbe_82598`, `ixgbe_82599`, `ixgbe_x540`, `ixgbe_x550`, `ixgbe_e610`. Each provides MAC-ops, PHY-ops, EEPROM-ops, link-ops.
- **`IxgbeAdapter<Phase>` typed lifecycle.** `Probed → HwInit → Online`.
- **Typed PF/VF mailbox.** Each msg as a typed struct; PF↔VF dispatch via exhaustive match.
- **Per-RX-ring typed.** Including AF_XDP-bound rings as a distinct ring variant.
- **DCB QoS configuration typed.** Per-TC bandwidth + PFC bitmap as typed structs.
- **IPsec SA install typed.** Per-direction (rx/tx) typed SA descriptor.
- **AdminQ for E610** — same pattern as ice driver (covered separately); E610 brings 10G into the modern Intel admin-queue world.

Grsec is mandatory. ixgbe has had CVE history (CVE-2017-5715 type, multiple SR-IOV-related bugs, mailbox parsing bugs in `ixgbe_mbx.c`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ixgbe_adapter` | per-netdev priv | `IxgbeAdapter<Phase>` |
| `struct ixgbe_hw` | HW abstraction | `IxgbeHw<S: Silicon>` |
| `struct ixgbe_ring` | per-direction ring | `Ring<Direction>` |
| `struct ixgbe_q_vector` | per-MSI-X vector NAPI | `QVector` |
| `ixgbe_probe(pdev, ent)` / `_remove(pdev)` | PCI probe/remove | `IxgbeAdapter::probe` / `Drop` |
| `ixgbe_open(netdev)` / `_close(netdev)` | netdev open/close | `IxgbeAdapter::open` / `close` |
| `ixgbe_xmit_frame(skb, netdev)` | TX | `IxgbeAdapter::xmit` |
| `ixgbe_clean_rx_irq(q_vector, rx_ring, budget)` / `_clean_tx_irq(...)` | NAPI poll | `QVector::poll` |
| `ixgbe_msix_clean_rings(irq, data)` | MSI-X handler | `Irq::msix_handler` |
| `ixgbe_init_hw_generic(hw)` / `_82598_*` / `_82599_*` / `_X540_*` / `_X550_*` / `_E610_*` | per-silicon init | `Silicon::init_hw` |
| `ixgbe_setup_link_*` / `_check_link_*` | link mgmt | `Link::setup` / `check` |
| `ixgbe_set_rx_mode(netdev)` / `_set_mac(netdev, p)` | filter mgmt | `Filter::set_rx_mode` / `_set_mac` |
| `ixgbe_configure_rss(adapter)` / `_set_rxfh(...)` | RSS | `Rss::configure` |
| `ixgbe_sriov_configure(pdev, num_vfs)` / `_disable_sriov(...)` | SR-IOV bring-up | `Sriov::enable` / `disable` |
| `ixgbe_vfdev_alloc(...)` / `_init_vfdev(...)` | per-VF init | `Sriov::init_vf` |
| `ixgbe_msg_task(adapter)` / `_rcv_msg_from_vf(...)` / `_set_vf_*` | VF mailbox | `VfMbx::*` |
| `ixgbe_fcoe_enable(netdev)` / `_disable(netdev)` / `_ddp_get(...)` / `_ddp_put(...)` | FCoE | `Fcoe::*` |
| `ixgbe_dcb_hw_config(hw, ...)` / `_setup_pfc(...)` | DCB | `Dcb::config` |
| `ixgbe_ipsec_offload_init(adapter)` / `_add_sa(...)` / `_del_sa(...)` | IPsec inline | `Ipsec::*` |
| `ixgbe_ptp_init(adapter)` / `_release(...)` | PTP | `Ptp::*` |
| `ixgbe_xsk_pool_setup(adapter, pool, qid)` / `_xsk_wakeup(...)` | AF_XDP | `Xsk::*` |
| `ixgbe_xdp(netdev, xdp)` / `_xdp_xmit(...)` | XDP | `Xdp::*` |
| `ixgbe_fw_update(...)` (E610) | FW update | `FwUpdate::run` |
| `ixgbe_devlink_register(...)` | devlink | `Devlink::register` |
| `ixgbe_aer_*` | AER | `Aer::*` |
| `ixgbe_io_error_detected` / `_io_slot_reset` / `_io_resume` | AER handlers | `Aer::*` |
| `ixgbe_get_drvinfo(...)` / `_get_ringparam(...)` / `_get_coalesce(...)` | ethtool | `Ethtool::*` |
| `ixgbe_mbx_*` | mailbox primitives | `Mbx::*` |
| `ixgbe_phy_init_ops` / `ixgbe_validate_phy_addr(...)` | PHY | `Phy::*` |

## Compatibility contract

REQ-1: PCI ID table — every ixgbe silicon: 82598EB, 82599, X540, X550, X552, X553, X550EM_a, X550EM_x, E610. Frozen.

REQ-2: Per-silicon HW ops — frozen against silicon datasheets. MAC/PHY/EEPROM/link ops differ per silicon.

REQ-3: SR-IOV ABI — PF→VF mailbox protocol frozen against `ixgbevf` consumer; covered messages: VF_INIT, VF_GET_MAC, VF_SET_MAC, VF_GET_VLAN, VF_SET_VLAN, VF_SET_MULTICAST, VF_API_NEGOTIATE, VF_RESET.

REQ-4: DCB ABI — PFC, ETS per IEEE 802.1Qaz; CEE compat preserved.

REQ-5: FCoE — DDB (Direct Data Placement) acceleration via FCoE engine; per-XID DDP context. Used with libfcoe + open-fcoe target.

REQ-6: IPsec — AES-GCM ESP offload per RFC 4106 + AH per RFC 4302. SA install via XFRM netdev hooks.

REQ-7: PTP — PHC clock; per-packet TX/RX timestamping.

REQ-8: AF_XDP zerocopy — zerocopy on RX into a UMEM-backed pool; TX from UMEM.

REQ-9: XDP — XDP_DRV native + XDP_REDIRECT.

REQ-10: AER — pci_error_handlers; recovery via reset.

REQ-11: E610 admin queue — newer silicon uses ice-derived admin queue protocol for FW interaction. Frozen against the admin queue ABI.

REQ-12: FW update (E610) — via devlink-flash. PSP-signed FW image.

REQ-13: Ethtool ops — full surface including PHY register dump, EEPROM, SFP/QSFP module access, fec.

## Acceptance Criteria

- [ ] AC-1: 82599 (10G SFP+) probes; link comes up; `iperf3` saturates 10G link.
- [ ] AC-2: X550 (10GBase-T) auto-negotiates 1/2.5/5/10G with various peer NICs.
- [ ] AC-3: E610 (modern 10/25G): probes; admin-queue init succeeds; FW version reported.
- [ ] AC-4: SR-IOV: 16 VFs enabled; per-VF MAC assigned via PF; KVM guest with vfio-pci uses VF; per-VF traffic isolated.
- [ ] AC-5: DCB+FCoE on a lossless fabric: PFC enabled; FCoE traffic prioritized; storage IO line-rate.
- [ ] AC-6: IPsec offload: XFRM SA installed with offload flag; per-packet IPsec encryption observable via per-port counter.
- [ ] AC-7: AF_XDP zerocopy: xdpsock benchmark hits 10M pps on single CPU; zero memcpy.
- [ ] AC-8: XDP_DRV: bpf program attached at NAPI; XDP_PASS / XDP_DROP / XDP_TX / XDP_REDIRECT all work.
- [ ] AC-9: PTP: ptp4l syncs to NIC PHC; offset stable within ±100ns.
- [ ] AC-10: AER injection: cleanly recovers.
- [ ] AC-11: FW update (E610) via devlink-flash: signed image accepted; unsigned refused.

## Architecture

**Per-silicon HW abstraction.** Each silicon family has its own ops table (`ixgbe_hw_ops`, `ixgbe_mac_ops`, `ixgbe_phy_ops`, `ixgbe_eeprom_ops`, `ixgbe_link_ops`). Cross-silicon helpers live in `ixgbe_common.c`. Driver dispatches via the `hw->mac.ops.X()` indirection.

**Standard Intel netdev pattern.** Per-CPU NAPI + per-CPU RX/TX rings + MSI-X vector. Dynamic interrupt moderation. Page-pool for RX buffer alloc. Standard offload set: TSO, GSO, GRO, RSS, RX/TX checksum.

**SR-IOV.** PF reserves resources for VFs at SR-IOV enable; per-VF MAC/VLAN/trust/rate via PF. Each VF receives a mailbox-delivered config. `ixgbevf` runs in guests.

**PF↔VF mailbox.** Doorbell-driven; PF and VF each have a mailbox; messages: typed structs with opcode + payload. PF side dispatches by opcode.

**DCB.** PFC (Priority Flow Control) + ETS (Enhanced Transmission Selection). HW-offloaded; driver configures per-TC bandwidth + PFC bitmap. Used for FCoE lossless.

**FCoE.** DDP (Direct Data Placement) acceleration: FCoE engine writes payload directly into pre-registered host BO + skips host CPU; used by libfcoe initiator + targets.

**IPsec inline.** SA descriptor installed into HW SA table; HW per-packet encrypts/decrypts. Per-port SA table capped (silicon-dependent).

**AF_XDP zerocopy.** RX descriptor points directly into a UMEM-backed page. NAPI fills descriptors from the user's UMEM ring; on completion, the descriptor is returned via UMEM completion ring. No memcpy.

**E610 admin queue.** Newer silicon brings ice-derived admin queue: SQ + CQ DMA-mapped; FW posts events + responses via CQ. Admin queue is the way to drive FW (config, FW update, FW reset) — separate from the IO path.

**FW update (E610).** devlink-flash → driver chunks the FW image, sends per-chunk via admin queue, FW validates signature and burns flash. Reboot or reset required to activate.

## Hardening

- All silicon ops indirect calls go through const-after-init ops tables.
- PF↔VF mailbox payload validated against opcode-specific schema.
- SR-IOV resource alloc validated against PF caps.
- IPsec SA install validates key length + algorithm.
- FCoE DDP context count capped per FW.
- AF_XDP ring sizes capped.
- AER recovery rate-limited.
- E610 admin-queue cmd timeout enforced.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — ethtool ingress, devlink-flash chunks, sysfs writes bounded copy_from_user.
- **PAX_KERNEXEC** — netdev_ops, ethtool_ops, per-silicon HW op tables, mbx dispatch, admin-queue dispatch (E610), IPsec ops, DCB ops, AER handlers all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — ethtool / netlink / sysfs / devlink entry paths per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `ixgbe_adapter` refcount, per-q_vector refcount, per-ring refcount, per-VF refcount, IPsec SA refcount, FCoE DDP context refcount saturating.
- **PAX_MEMORY_SANITIZE** — IPsec SA key material `kfree_sensitive`. PHC clock samples cleared. Per-VF context cleared on VF disable. FCoE DDP buffer (containing in-flight storage data) cleared on free.
- **PAX_UDEREF** — SMAP/SMEP on ethtool, netlink, devlink, sysfs.
- **PAX_RAP / kCFI** — netdev_ops, ethtool_ops, per-silicon op tables, mbx dispatch, IPsec offload ops, DCB ops, AER handlers, XDP ops all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/ixgbe/<bdf>/*`) reveals ring pointers + admin-queue addresses + IPsec SA table; scrubbed for non-root.
- **GRKERNSEC_DMESG** — ixgbe verbose logs (per-VF state changes revealing tenant topology, mbx message trace) restricted from non-root.
- **CAP_NET_ADMIN required on every state-changing op** — ethtool MAC/RSS/coalesce/ring, SR-IOV enable, devlink-flash, IPsec SA install via XFRM netdev, FCoE enable/disable.
- **PF→VF mailbox per-opcode allowlist + size strict** — PF refuses unknown opcodes; VF requests for PF-scope ops (e.g. set-promisc) refused. Closes "compromised VF tricks PF into reconfig" class.
- **VF spoof-check enforced** — egress ACL drops VF-sent packets with non-assigned source MAC (HW-enforced). Closes cross-VF MAC spoofing class.
- **IPsec SA key material kfree_sensitive** — SA keys cleared on free; closes key-leak across SA reassignment.
- **FCoE DDP buffer per-XID scoped** — DDP buffers tagged with owning process; cross-process access refused. Closes a path where one tenant's FCoE traffic could land in another's DDP buffer.
- **FW image signature mandatory (E610)** — Intel-signed; LOCKDOWN_INTEGRITY_MAX disables FW updates.
- **PF resource cap enforced per-VF** — refuses to allocate more queues/RSS-slots/IPsec-SAs than the per-VF quota.
- **PHY MDIO access scoped** — direct PHY register access via ethtool requires CAP_NET_ADMIN; some destructive PHY ops require CAP_SYS_RAWIO.
- **PTP clock-adjust CAP_SYS_TIME** — not CAP_NET_ADMIN; matches PTP semantics.
- **AER recovery rate-limit** — 3 per 60s; closes "Intel-side AER storm" class.
- **DCB-config rate-limit** — DCB config changes rate-limited to prevent NIC bring-down storms.
- **Debugfs CAP_SYS_ADMIN** — all writes; reads scrub addresses.
- **AF_XDP UMEM access strict** — UMEM pages held with mmu-notifier; on user munmap, GPU mappings (er, NIC DMA mappings) invalidated.
- **DCB CEE legacy mode CAP_SYS_ADMIN** — CEE (older DCB protocol) install gated; closes legacy-protocol attack.

Per-doc rationale: ixgbe is in the installed base everywhere — every datacenter, every BMC out-of-band network, every enterprise server pre-2020. The huge multi-silicon code surface (82598 through E610 spans 16 years of silicon revisions) is a major surface for code-quality bugs. The PF↔VF mailbox is the primary cross-tenant-isolation surface; bugs there have historically been an SR-IOV escape vector. The Rust translation's per-silicon `IxgbeSilicon` trait + typed PF↔VF mailbox close the dominant bug classes structurally. IPsec inline offload key material kfree_sensitive closes key-leak across SA reassignment. FCoE DDP per-XID scoping closes cross-tenant FCoE-data class.

## Open Questions

- [ ] Q1: 82598EB support — 16-year-old silicon, still deployed in long-life embedded + industrial. Maintain? Recommendation: maintain via the per-silicon trait, deprecation warning at probe.
- [ ] Q2: E610 — newest silicon, uses ice-derived admin queue. Share admin-queue code with the `ice` driver (separate Tier-3) or duplicate? Recommendation: shared admin-queue crate consumed by both.
- [ ] Q3: ixgbevf (sibling VF driver, runs in guests) — separate Tier-3 needed? Recommendation: yes; ixgbevf is its own driver in upstream.
- [ ] Q4: DCB CEE legacy — keep or drop? Recommendation: keep, with deprecation warning.

## Verification

- **Kani SAFETY**: prove silicon-trait dispatch is exhaustive; PF→VF mailbox payload validation per-opcode.
- **TLA+**: model SR-IOV PF/VF mailbox concurrent paths; check no deadlock + no PF receiving inconsistent state from a VF.
- **Verus**: functional spec of IPsec SA install — for valid SA params, produces HW descriptor matching XFRM SA; for invalid (key length, algo), returns -EINVAL.
- **Kani+Verus**: invariant that every active VF has tracked per-VF state; cleanup at disable_sriov removes all.
- **Integration**: full ixgbe matrix — 82599, X540, X550, E610 each probed + line-rate iperf3 + SR-IOV stress (64 VFs across PF) + DCB+FCoE smoke + IPsec offload + AF_XDP zerocopy + XDP_DRV. PTP sync. AER injection. FW update on E610.
- **Fuzz**: PF→VF mailbox fuzzer; admin-queue response fuzzer (E610); ethtool ioctl fuzzer.
- **Penetration**: from compromised VF, attempt to send PF-scope mailbox cmd — refused. From unprivileged container, attempt SR-IOV enable — refused (CAP_NET_ADMIN).

## Out of Scope

- ixgbevf (VF driver, runs in guests) — separate Tier-3
- ice (Intel 100G driver, sibling) — covered as `intel-ice.md` (pre-existing)
- i40e (40G XL710, sibling) — separate Tier-3 (next iteration target)
- e1000e (1G, older) — separate Tier-3
- libfcoe + open-fcoe target — separate net stack docs
