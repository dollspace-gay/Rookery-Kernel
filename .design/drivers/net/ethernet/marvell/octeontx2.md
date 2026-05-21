# Tier-3: drivers/net/ethernet/marvell/octeontx2/{af,nic}/ — Marvell OcteonTX2/CN10K/CN20K SoC NIC (CN9xxx 64-core ARM SoC + CN10Kxx 96-core SmartNIC + CN20Kxx 144-core DPU — DPU class for cloud service providers)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
upstream-paths:
  - drivers/net/ethernet/marvell/octeontx2/af/                   (~35000 lines: AF — Admin Function, host-side root PF of the SoC NIC complex)
  - drivers/net/ethernet/marvell/octeontx2/af/rvu.c              (~3500 lines: RVU — Resource Virtualization Unit, sub-PF/VF allocator)
  - drivers/net/ethernet/marvell/octeontx2/af/rvu_nix.c          (~5000 lines: NIX — Network Interface eXchange, RX/TX queues)
  - drivers/net/ethernet/marvell/octeontx2/af/rvu_npa.c          (NPA — Network Pool Allocator, buffer pool mgr)
  - drivers/net/ethernet/marvell/octeontx2/af/rvu_npc.c          (~5500 lines: NPC — Network Packet Classifier, programmable parser + flow steering)
  - drivers/net/ethernet/marvell/octeontx2/af/rvu_cpt.c          (CPT — Crypto Processor, IPsec offload)
  - drivers/net/ethernet/marvell/octeontx2/af/cgx.c              (CGX — Common Glue, MAC + PHY interfaces 10/25/40/100G)
  - drivers/net/ethernet/marvell/octeontx2/af/rpm.c              (RPM — Reset & Power Management for CN10K)
  - drivers/net/ethernet/marvell/octeontx2/af/mcs.c              (MACsec engine)
  - drivers/net/ethernet/marvell/octeontx2/af/mbox.c             (PF↔VF mailbox)
  - drivers/net/ethernet/marvell/octeontx2/af/ptp.c              (PHC)
  - drivers/net/ethernet/marvell/octeontx2/af/cn20k/             (CN20K-specific extensions)
  - drivers/net/ethernet/marvell/octeontx2/nic/                  (~29000 lines: per-VF/PF netdev driver)
  - drivers/net/ethernet/marvell/octeontx2/nic/otx2_pf.c         (PF netdev)
  - drivers/net/ethernet/marvell/octeontx2/nic/otx2_vf.c         (VF netdev)
  - drivers/net/ethernet/marvell/octeontx2/nic/otx2_common.c     (shared)
  - drivers/net/ethernet/marvell/octeontx2/nic/cn10k.c           (CN10K silicon specifics)
  - drivers/net/ethernet/marvell/octeontx2/nic/cn20k.c           (CN20K silicon specifics)
  - drivers/net/ethernet/marvell/octeontx2/nic/cn10k_ipsec.c     (IPsec offload integration)
  - drivers/net/ethernet/marvell/octeontx2/nic/cn10k_macsec.c    (MACsec offload integration)
  - drivers/net/ethernet/marvell/octeontx2/nic/otx2_tc.c         (TC HW-offload)
  - drivers/net/ethernet/marvell/octeontx2/nic/qos.c             (QoS / hierarchical scheduling)
  - drivers/net/ethernet/marvell/octeontx2/nic/qos_sq.c          (per-class SQ)
-->

## Summary

octeontx2 is the driver for Marvell's OcteonTX2 / CN10K / CN20K family of ARM-based SoC NICs — a class of silicon distinct from standard NIC HBAs. These are full ARM-Cortex-A57/A72/A78 SoCs with embedded 25-100G Ethernet, used as:

- **SmartNICs** — PCIe-attached cards with a SoC + NIC running an embedded Linux OS that offloads networking work from the host. Used by cloud service providers (Cloudflare, AWS Outposts, etc.) for accelerated firewall, load balancer, custom packet processing.
- **DPUs** (Data Processing Units) — CN10K + CN20K class. ARM SoC + NIC + crypto + storage offload + accelerators. Replaces some host-CPU workload entirely. Competitor to NVIDIA BlueField + Intel IPU.
- **Embedded ARM platforms** — CN9xxx in routers (Cisco, Juniper edge), security appliances, NAS.

The driver is unique architecturally: the SoC is presented as one PCI device but contains many internal sub-functions accessed via a hierarchical SR-IOV + RVU (Resource Virtualization Unit) model. Components:

- **AF (Admin Function)** — the root host-side controller. Manages all sub-resources via mailbox. Located in `af/`. ~35K LOC.
- **NIC PF/VF** — per-PF/per-VF netdev drivers in `nic/`. Each sub-function gets its own netdev.
- **NIX / NPA / NPC / CPT / CGX / RPM / MCS** — internal SoC blocks each with their own driver code in `af/`.

Total ~64K LOC.

## Rust translation posture

octeontx2 has a unique architecture among Linux NIC drivers — a multi-block SoC managed by a root AF + per-block per-PF/VF drivers. Translation:

- **`octeontx2-af` crate** — root admin function. Owns mailbox, RVU resource allocator, per-block subdrivers.
- **`octeontx2-nic` crate** — per-PF/VF netdev driver. Talks to AF via mailbox.
- **`octeontx2-rvu` crate** — RVU resource allocator. Per-block (NIX/NPA/NPC/CPT/CGX/RPM/MCS) resource carve-out.
- **Per-block sub-crates**: `octeontx2-nix` (RX/TX queues), `octeontx2-npc` (programmable parser), `octeontx2-cpt` (crypto offload), `octeontx2-cgx` (PHY), `octeontx2-mcs` (MACsec).
- **Typed mailbox protocol.** Hundreds of mailbox messages between AF + PF/VF; typed per-msg.
- **Per-silicon types.** OcteonTX2 (CN9xxx), CN10K, CN20K each have variant blocks.

The grsec/PaX section is mandatory. OcteonTX2 has unique DPU concerns (cross-tenant isolation between many VFs on one SoC), and the SmartNIC use case adds a new trust boundary (the host CPU may not fully trust the SmartNIC SoC, and vice versa).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rvu` | RVU resource allocator | `Rvu` |
| `struct rvu_pfvf` | per-PF/VF in RVU | `RvuPfvf` |
| `struct otx2_nic` | per-netdev NIC priv | `OtxNic<Phase>` |
| `struct otx2_hw` | HW abstraction | `OtxHw<Silicon>` |
| `struct mbox` | mailbox state | `Mbox` |
| `struct cgx` | CGX port | `Cgx` |
| `struct nix_hw` | NIX block | `Nix` |
| `struct npa_pool` | NPA buffer pool | `NpaPool` |
| `struct cpt_inst` | CPT crypto instance | `CptInst` |
| `rvu_probe(pdev, ent)` / `_remove(pdev)` | AF PCI probe/remove | `Rvu::probe` / `Drop` |
| `rvu_setup_hw_resources(rvu)` | RVU init | `Rvu::setup` |
| `rvu_check_rsrc_availability(...)` / `_alloc_rsrc(...)` | resource alloc | `Rvu::alloc_rsrc` |
| `mbox_send_msg(...)` / `_recv_msg(...)` / `_init(...)` | mailbox | `Mbox::*` |
| `cgx_lmac_init(...)` / `_lmac_link_*` | CGX init + link mgmt | `Cgx::*` |
| `rvu_nix_init(rvu)` / `_nix_*_msg_handler(...)` | NIX setup + mbox handlers | `Nix::*` |
| `rvu_npa_init(rvu)` / `_npa_alloc_pool(...)` | NPA setup | `Npa::*` |
| `rvu_npc_init(rvu)` / `_npc_install_flow(...)` / `_npc_kpu_program(...)` | NPC setup + KPU (Key Processing Unit) programming | `Npc::*` |
| `rvu_cpt_init(rvu)` / `_cpt_*_msg_handler(...)` | CPT setup | `Cpt::*` |
| `rpm_*` (CN10K reset/power) | RPM ops | `Rpm::*` |
| `mcs_*` | MACsec engine | `Mcs::*` |
| `otx2_probe(pdev, ent)` / `_remove(pdev)` | NIC PF probe | `OtxNic::probe_pf` |
| `otx2vf_probe(...)` / `_remove(...)` | NIC VF probe | `OtxNic::probe_vf` |
| `otx2_open(netdev)` / `_close(netdev)` | netdev open/close | `OtxNic::open` |
| `otx2_xmit(skb, netdev)` | TX | `OtxNic::xmit` |
| `otx2_poll(napi, budget)` | NAPI poll | `Napi::poll` |
| `otx2_msg_handler(work)` | mailbox event dispatch from AF | `Mbox::on_event` |
| `otx2_tc_setup_flower(...)` | TC HW-offload | `Tc::setup_flower` |
| `otx2_set_rss_table(...)` | RSS | `Rss::set_table` |
| `cn10k_ipsec_*` | IPsec inline offload | `Ipsec::*` |
| `cn10k_macsec_*` | MACsec offload | `Macsec::*` |
| `otx2_ptp_init(...)` | PHC | `Ptp::init` |
| `otx2_qos_*` | hierarchical QoS scheduling | `Qos::*` |

## Compatibility contract

REQ-1: PCI ID table — CN9xxx (OcteonTX2), CN10Kxx (CN10K), CN20Kxx (CN20K). Frozen.

REQ-2: AF mailbox ABI — frozen against `mbox.h` (hundreds of messages between AF + PF/VF subfunctions).

REQ-3: RVU resource model — per-block resource allocator. Each PF/VF gets carved resources (NIX queues, NPA pools, NPC flow entries, CPT inflights).

REQ-4: NIX queue model — per-VF RQ/SQ/CQ. SQ has scheduler element for QoS.

REQ-5: NPA pool model — per-VF buffer pools shared via NIX. Auto-fill via NPA.

REQ-6: NPC programmable parser — Key Processing Unit (KPU) programmed with parser rules; flow steering programmed via flow-install mbox.

REQ-7: CPT crypto offload — IPsec ESP/AH inline offload; per-SA install via mbox.

REQ-8: MCS MACsec engine — per-port MACsec session install.

REQ-9: CGX PHY interfaces — 10/25/40/50/100G Ethernet per port; SerDes config.

REQ-10: SR-IOV — each PF can enable VFs; per-VF carved resources via AF mbox.

REQ-11: TC HW-offload via NPC flow rules.

REQ-12: PTP — strong PHC; per-packet TS.

REQ-13: QoS — hierarchical scheduling (CBS + TAS + custom shaper) at NIX SQ level.

REQ-14: FW image — Marvell-signed.

REQ-15: Per-silicon (CN9/CN10/CN20) feature differences — CN10K added CPT inline IPsec, MCS MACsec, RPM; CN20K adds further extensions.

## Acceptance Criteria

- [ ] AC-1: CN10K SmartNIC probes; AF init; PF/VF netdevs appear.
- [ ] AC-2: 100G iperf3 sustained; per-CPU IRQ distribution.
- [ ] AC-3: SR-IOV 64 VFs per PF; per-VF resource quota; iavf-equivalent in VF guest works.
- [ ] AC-4: TC HW-offload: flower rule installed in NPC.
- [ ] AC-5: IPsec inline (CN10K): XFRM SA installed via mbox; per-packet offload observable.
- [ ] AC-6: MACsec inline (CN10K): per-port MKA-managed MACsec; encrypted frames.
- [ ] AC-7: NPC KPU custom parser load: custom flow rules using vendor parser.
- [ ] AC-8: QoS hierarchical: per-class shaping with traffic shaping at SQ.
- [ ] AC-9: PTP sync sub-100ns.
- [ ] AC-10: FW image update via devlink-flash: signed image accepted.

## Architecture

**AF + RVU + per-block layering.** AF is the root admin function on PF0 of the SoC NIC complex. It boots first, initializes the RVU resource allocator, brings up each block (NIX, NPA, NPC, CPT, CGX, RPM, MCS). Per-PF/VF netdev drivers communicate with AF via mailbox to request resources + configure flow rules + install crypto SAs.

**Mailbox protocol.** AF↔PF/VF mailbox is DMA-mapped + doorbell-driven. Hundreds of message types covering every config knob. Per-VF mailboxes are scoped to that VF's resources.

**NIX queues.** Per-VF NIX RQ (Receive Queue) + SQ (Send Queue) + CQ (Completion Queue). SQ has a scheduler element programmable for hierarchical QoS.

**NPC programmable parser.** Marvell's distinguishing feature: hardware parser is programmable via KPU (Key Processing Unit). New protocols can be parsed without driver code changes — vendor ships KPU programs. NPC flow table installed via mbox.

**CPT crypto offload.** Per-VF CPT inflight context. IPsec ESP/AH SAs installed per-VF; per-packet ESP/AH inline; same for MACsec.

**SmartNIC + DPU.** When configured as SmartNIC/DPU, the SoC runs an embedded Linux (managed separately). Host PF talks to the SoC via the mailbox; PFs presented to the host represent SoC-side processing.

**Per-silicon variant.** CN9xxx is the OcteonTX2 base; CN10K adds inline IPsec + MACsec + RPM; CN20K extends with newer NIX/NPA features + 144-core ARM cluster.

## Hardening

- AF mailbox auth: each mailbox message authenticated against sender PF/VF ID.
- RVU resource alloc strict: cannot over-allocate.
- NPC KPU programs signed (vendor) before load.
- Per-VF mbox message validated against VF's resource scope.
- CPT inflight count capped.
- CGX PHY config bounded.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — netlink, sysfs, devlink, tc-flower ingress bounded.
- **PAX_KERNEXEC** — AF op tables, mailbox dispatch, RVU resource allocator, per-block op tables, NPC parser dispatch, CPT cmd dispatch, mac80211/netdev ops, MACsec ops, IPsec ops, AER handlers all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — every user-facing entry path per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — RVU refcount, per-PF/VF refcount, per-NIX-queue refcount, per-NPA-pool refcount, per-CPT-SA refcount, per-MCS-session refcount saturating.
- **PAX_MEMORY_SANITIZE** — IPsec/MACsec key material `kfree_sensitive`. NPC KPU program load buffer cleared. CPT cmd DMA buffers cleared. Per-VF mbox scratch cleared.
- **PAX_UDEREF** — SMAP/SMEP on every user-facing entry.
- **PAX_RAP / kCFI** — AF ops, mailbox dispatch, per-block ops, NPC KPU dispatcher, CPT cmd dispatch, MACsec ops, IPsec ops all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs scrubbed (RVU + per-block state).
- **GRKERNSEC_DMESG** — octeontx2 verbose logs (per-VF state, NPC flow entries, mbox trace revealing tenant topology) restricted from non-root.
- **CAP_NET_ADMIN required** — every state-changing op: SR-IOV enable, tc-flower install, IPsec SA install, MACsec MKA session.
- **AF-to-PF/VF mailbox per-msg allowlist + size strict** — per-VF mbox messages validated against VF's resource scope (a VF cannot request resources belonging to another VF). Closes 'compromised VF tricks AF into reconfig of sibling VF' SR-IOV-escape class.
- **RVU resource cap strict** — each PF/VF cannot allocate more than its quota; closes resource-exhaustion DoS.
- **NPC KPU program signature mandatory** — KPU programs are Marvell-signed; unsigned refused. Closes 'compromised KPU program causes wrong packet parsing → flow-rule bypass' class. LOCKDOWN_INTEGRITY_MAX disables KPU updates.
- **NPC flow-rule install per-VF scoped** — VF can only install flow rules targeting its own NIX queues; cannot redirect cross-VF traffic. Closes cross-tenant traffic snoop class.
- **CPT inline IPsec key material kfree_sensitive** — after install to CPT, host-side key memory cleared.
- **MCS MACsec key material kfree_sensitive** — same.
- **PTP CAP_SYS_TIME** — clock adjust requires CAP_SYS_TIME.
- **DPU/SmartNIC trust boundary explicit** — when configured as DPU, host PF cannot trust SoC-side processing; AF responses to host-side queries validated against expected schemas. Conversely, SoC-side cannot trust host-supplied flow rules — validated against per-VF scope.
- **CGX/RPM PHY-register access scoped** — direct PHY/serdes reg access via debugfs CAP_SYS_ADMIN.
- **AF recovery rate-limit** — AF reset rate-limited; closes 'compromised AF gaslights host into endless reset'.
- **MACsec replay window enforced** — anti-replay window per RFC; closes replay attacks.
- **Per-VF mbox queue rate-limit** — closes 'VF spams AF with mbox requests → AF DoS'.
- **CGX link state authoritative from AF** — VFs cannot fake link state to userspace; closes 'VF reports link-up for traffic injection while link is actually down' class.

Per-doc rationale: octeontx2 is a unique driver architecturally — a multi-block SoC NIC presented as a hierarchy of PFs/VFs with internal blocks (NIX/NPA/NPC/CPT/CGX/MCS). The headline grsec concern is cross-tenant isolation at scale (CN20K supports many tens of VFs on one SoC) + the SmartNIC/DPU trust boundary (host-CPU vs SoC-CPU). Defenses focus on the AF mailbox as the central authorization point: every PF/VF request validated against the requester's resource scope. NPC KPU program signature is unique: programmable hardware parsers introduce a new attack surface where malicious parser programs could bypass flow rules — closed by mandatory Marvell signature + LOCKDOWN integration. Inline IPsec/MACsec key kfree_sensitive after CPT/MCS install is critical because keys live in silicon (host-side never retains plaintext).

## Open Questions

- [ ] Q1: SmartNIC/DPU mode — when in DPU mode, host PF + SoC-side both run drivers. Need explicit design for the host↔SoC trust boundary.
- [ ] Q2: NPC KPU program updates — how often does Marvell push new KPU programs? Maintain a per-version KPU table?
- [ ] Q3: CN20K newer blocks (CPT 2.0, NIX next-gen) — track new feature additions.
- [ ] Q4: Embedded ARM platforms (CN9xxx in Cisco/Juniper) — different deployment posture vs SmartNIC; document separately.
- [ ] Q5: QoS scheduling — hierarchical scheduling has many edge cases; verus-spec the scheduler tree manipulation.

## Verification

- **Kani SAFETY**: prove AF mbox dispatcher validates source PF/VF against requested resource scope. Prove NPC KPU program signature check cannot be bypassed.
- **TLA+**: model SR-IOV PF/VF mbox concurrency + RVU resource allocator; check no resource leak / over-allocation across any sequence.
- **Verus**: functional spec of NPC flow-rule install — for valid rule scoped to VF, installs at HW; cross-VF refused.
- **Kani+Verus**: invariant that every active VF has tracked resources; cleanup at disable_sriov removes all.
- **Integration**: CN10K SmartNIC + DPU smoke (host PF + SoC-side Linux); SR-IOV 64-VF stress; TC HW-offload + IPsec + MACsec all concurrent; PTP sync; FW update.
- **Fuzz**: AF mbox msg fuzzer (mutate every msg from a VF — AF must validate scope); NPC KPU program fuzzer.
- **Penetration**: compromised VF attempts AF mbox msg targeting sibling VF's resources — refused. Unsigned KPU program — refused.

## Out of Scope

- OcteonTX2 SoC-side Linux (embedded) — out of host-driver scope
- DPU programming model — separate ecosystem doc
- liquidio (older Marvell SoC NIC) — separate Tier-3
- Marvell other NICs (mvneta, mvpp2, mvgbe) — separate Tier-3
- prestera (Marvell switch) — separate Tier-3
- NPC KPU program development — Marvell-side
