# Tier-3: drivers/net/ethernet/intel/iavf/{iavf_main,iavf_virtchnl,iavf_common,iavf_adminq,iavf_txrx,iavf_ethtool,iavf_fdir,iavf_adv_rss,iavf_ptp}.c — Intel Adaptive VF (VF-side driver for i40e + ice + future Intel SR-IOV silicon, runs in KVM/QEMU/Hyper-V guests)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
sibling-pf:
  - drivers/net/ethernet/intel/i40e.md
  - drivers/net/ethernet/intel/intel-ice.md (pre-existing)
upstream-paths:
  - drivers/net/ethernet/intel/iavf/iavf_main.c                  (~5500 lines: netdev ops, NAPI, RX/TX, MSI-X, init flow)
  - drivers/net/ethernet/intel/iavf/iavf_virtchnl.c              (~2955 lines: VirtChnl VF-side msg send + response handling)
  - drivers/net/ethernet/intel/iavf/iavf_common.c                (~1700 lines: HW utilities shared with PF drivers)
  - drivers/net/ethernet/intel/iavf/iavf_adminq.c                (VF AdminQ — mailbox to PF)
  - drivers/net/ethernet/intel/iavf/iavf_txrx.c                  (~3500 lines: descriptor RX/TX fast path)
  - drivers/net/ethernet/intel/iavf/iavf_ethtool.c               (~1300 lines: ethtool ops — limited subset vs PF)
  - drivers/net/ethernet/intel/iavf/iavf_fdir.c                  (~925 lines: Flow Director — VF flow rules)
  - drivers/net/ethernet/intel/iavf/iavf_adv_rss.c               (advanced RSS — per-header-field config)
  - drivers/net/ethernet/intel/iavf/iavf_ptp.c                   (PTP per-VF)
  - drivers/net/ethernet/intel/iavf/iavf.h
  - include/linux/avf/virtchnl.h                                  (VirtChnl ABI — shared with PF)
-->

## Summary

iavf is the Intel Adaptive Virtual Function driver — the guest-side driver that runs inside KVM/QEMU/VMware/Hyper-V/Xen virtual machines when they are presented with an Intel SR-IOV virtual function from i40e (XL710), ice (E810), or future Intel SR-IOV silicon. The name "adaptive" comes from the fact that iavf adapts to whichever PF generation it is connected to via runtime VirtChnl ABI negotiation — the same VF kernel module binary works against XL710 PF + E810 PF + future Intel PF without recompile.

Mechanically iavf is a thin netdev driver that delegates all privileged ops to the PF via VirtChnl messages. It has no direct hardware access (the SR-IOV VF presents only IO queues + a mailbox); everything else (MAC config, VLAN add, RSS config, link state) goes through the mailbox to the PF. The VF sees its assigned MSI-X vectors + per-CPU RX/TX rings; everything else is virtual.

iavf supports:
- Standard netdev RX/TX with NAPI.
- ethtool (limited subset: stats, ring, RSS, coalesce — no link, no NVM, no flash).
- AF_XDP zerocopy (when host PF enables it).
- XDP_DRV.
- VLAN filtering.
- Flow Director (Fdir) rules — limited to the VF's resource quota.
- Advanced RSS (per-header-field control).
- PTP (when host PF advertises).
- VirtChnl ABI version negotiation.

This Tier-3 covers ~16000 lines.

## Rust translation posture

iavf is the smaller, cleaner half of the SR-IOV pair (PF is more complex). Translation:

- **`iavf-core` crate** — netdev, NAPI, RX/TX, ethtool, MSI-X.
- **Shared `virtchnl-msg` crate** with i40e + ice for VirtChnl ABI types. iavf's responsibility is the VF-side message builders + response parsers.
- **IavfAdapter<Phase>** typed: `Probed → VfResetRequested → VirtChnlNegotiated → VfResourcesReceived → Online`.
- **VirtChnl ABI version negotiation typed.** `VirtChnlVer::{V1_0, V1_1}` with per-version op availability checked at compile time where possible.
- **Per-ring typed.** Standard pattern.
- **PF-as-adversary model.** Unique to VF drivers: the iavf VF must defensively validate PF responses + async events because the host PF (compromised or buggy) could send malformed VirtChnl messages.

Grsec is mandatory. The PF-as-adversary trust model is the headline. Historical CVEs in iavf have come from inadequate PF-message validation (CVE-2024-27059 PF response OOB read in VirtChnl handler, related class).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iavf_adapter` | per-VF state | `IavfAdapter<Phase>` |
| `struct iavf_ring` | per-direction ring | `Ring<Direction>` |
| `struct iavf_q_vector` | per-MSI-X NAPI | `QVector` |
| `struct iavf_vsi` | VF-side VSI (single) | `Vsi` |
| `iavf_probe(pdev, ent)` / `_remove(pdev)` | PCI probe/remove | `IavfAdapter::probe` / `Drop` |
| `iavf_init_module()` / `_exit_module(...)` | module load/unload | `Module::*` |
| `iavf_init_get_resources(adapter)` / `_get_capabilities(...)` | post-reset init via VirtChnl | `Init::get_resources` |
| `iavf_open(netdev)` / `_close(netdev)` | netdev open/close | `IavfAdapter::open` / `close` |
| `iavf_xmit_frame(skb, netdev)` | TX | `IavfAdapter::xmit` |
| `iavf_napi_poll(napi, budget)` | NAPI poll | `Napi::poll` |
| `iavf_msix_clean_rings(irq, data)` | MSI-X handler | `Irq::msix_handler` |
| `iavf_send_pf_msg(adapter, op, msg, len)` | VirtChnl msg send | `Vc::send_pf_msg` |
| `iavf_virtchnl_completion(adapter, op, v_retval, msg, msglen)` | VirtChnl response handler | `Vc::on_response` |
| `iavf_handle_reset_event(adapter)` | VF reset handler | `Reset::handle` |
| `iavf_set_mac(netdev, p)` / `_change_mtu(...)` / `_set_rx_mode(...)` | netdev cfg via VirtChnl | `Netdev::set_*` |
| `iavf_add_vlan(adapter, vid)` / `_del_vlan(adapter, vid)` | VLAN add/del | `Vlan::add` / `del` |
| `iavf_configure_rss(adapter)` / `_get_rxfh(...)` / `_set_rxfh(...)` | RSS | `Rss::*` |
| `iavf_request_traffic_irqs(adapter, basename)` | per-queue MSI-X req | `Irq::request_traffic` |
| `iavf_add_fdir_filter(...)` / `_del_fdir_filter(...)` | Flow Director | `Fdir::add` / `del` |
| `iavf_adv_rss_*` | advanced RSS | `AdvRss::*` |
| `iavf_ptp_init(adapter)` | PTP | `Ptp::init` |
| `iavf_xdp(netdev, xdp)` / `_xdp_xmit(...)` | XDP | `Xdp::*` |
| `iavf_xsk_pool_setup(...)` / `_xsk_wakeup(...)` | AF_XDP | `Xsk::*` |
| `iavf_get_drvinfo(...)` / `_get_ringparam(...)` / `_get_coalesce(...)` | ethtool | `Ethtool::*` |
| `iavf_negotiate_vc_version(adapter, vc_version)` | VirtChnl version negotiation | `Vc::negotiate_version` |

## Compatibility contract

REQ-1: PCI ID table — VF IDs from XL710 family (i40e PF), E810 family (ice PF), and future Intel SR-IOV silicon. Frozen as upstream grows.

REQ-2: VirtChnl ABI — `include/linux/avf/virtchnl.h`. Version negotiation via `VIRTCHNL_OP_VERSION` first message. Subsequent ops scoped to negotiated version.

REQ-3: VF init flow — `RESET_VF` requested via mailbox → wait for PF to complete → `GET_VF_RESOURCES` → receive caps (queues, MSI-X, VSI count, RSS table size, MTU, MAC) → configure rings → enable queues.

REQ-4: VF AdminQ — VF-side mailbox ring. Each VF gets its own mailbox; PF runs the other end. Async events from PF arrive on the ARQ.

REQ-5: Per-VF resource limit — VF's queues/MSI-X/RSS-slots/Fdir-rules bounded by PF-assigned quota (returned in GET_VF_RESOURCES). VF refuses to exceed.

REQ-6: VLAN — VF requests VLAN adds via VirtChnl; PF enforces (may reject if not on allowed-VLAN list).

REQ-7: MAC — VF can request MAC via VirtChnl; PF enforces trust-policy (untrusted VFs refused MAC change).

REQ-8: Reset — PF can reset the VF at any time (e.g. host admin re-config); VF receives `VIRTCHNL_EVENT_RESET_IMPENDING` async event; flushes state + re-inits.

REQ-9: Link state — VF doesn't poll link; PF posts `VIRTCHNL_EVENT_LINK_CHANGE` async event.

REQ-10: PTP — VF-side PTP works only if PF allows; PHC clock samples come via VirtChnl.

REQ-11: AF_XDP — VF can run AF_XDP zerocopy if PF supports + assigned dedicated queues.

REQ-12: XDP_DRV — native XDP works.

REQ-13: Fdir — VF Fdir rules installed via VirtChnl; subject to per-VF rule count quota.

REQ-14: Advanced RSS — per-header-field RSS config via VirtChnl (newer VC versions only).

REQ-15: Migration / live migrate — when host live-migrates the guest, VF disappears + reappears on the new host's PF; iavf handles via reset + re-init.

## Acceptance Criteria

- [ ] AC-1: KVM guest with vfio-pci passing through an XL710 VF: iavf probes; VirtChnl version negotiated; GET_VF_RESOURCES returns expected caps; `ip link` shows eth0 UP.
- [ ] AC-2: iperf3 within KVM guest: line-rate per VF's assigned queue count.
- [ ] AC-3: VLAN add: from guest, `ip link add link eth0 type vlan id 100`; VLAN tagged frames work.
- [ ] AC-4: MAC change refused: untrusted VF tries to change MAC; PF refuses; VF logs the refusal.
- [ ] AC-5: VF reset (PF-initiated): host runs `ip link set <pf> vf <n> mac aa:..`; PF resets VF; iavf inside guest re-inits; netdev recovers.
- [ ] AC-6: ethtool: stats, ring, coalesce work; link/NVM/flash refused (no VF access).
- [ ] AC-7: PTP: when PF allows, ptp4l in guest syncs to PHC.
- [ ] AC-8: AF_XDP zerocopy: xdpsock benchmark within guest hits the assigned queue's pps cap.
- [ ] AC-9: Fdir: install per-flow filter via ethtool; rule installed at PF; counter increments.
- [ ] AC-10: Live migrate: KVM guest moved between hosts; iavf in guest re-inits cleanly via reset path.
- [ ] AC-11: VirtChnl version cross-compat: iavf binary against XL710 PF + E810 PF + future PF all negotiate successfully.

## Architecture

**Init flow.** iavf init is driven by PF responses:
1. PCI probe → request RESET_VF via mailbox.
2. Poll for PF reset complete signal.
3. Send GET_VF_RESOURCES; receive VF caps.
4. Allocate rings + register MSI-X.
5. Send CONFIG_VSI_QUEUES + CONFIG_RSS_KEY + CONFIG_RSS_LUT.
6. Send ENABLE_QUEUES.
7. Netdev online.

If any step fails, retry up to N times; permanent failure → driver-failed state.

**VirtChnl transport.** AdminQ-equivalent at the VF: per-VF mailbox SQ + ARQ. SQ for VF→PF messages, ARQ for PF→VF responses + async events. Each msg has opcode + payload.

**PF-as-adversary defensive validation.** Every PF response + async event payload validated against expected schema before consumption. Mismatched sizes / unknown opcodes / out-of-range values dropped with rate-limited warn. Closes the historical CVE class where a buggy/compromised PF could trigger VF-side UB.

**Reset handling.** PF can reset VF at any time. iavf detects via async event or by polling reset register. On reset:
1. Stop NAPI.
2. Tear down rings.
3. Wait for PF to signal reset complete.
4. Re-init from step 3 of init flow.

**Live migration.** When KVM live-migrates the guest, the VF on the source host is detached + re-attached on the destination. iavf inside the guest sees a reset + re-init via the same flow. Migration latency dominated by re-init time.

**Per-CPU NAPI.** Standard Intel pattern: per-MSI-X q_vector + NAPI; dynamic interrupt moderation.

**Fdir + advanced RSS.** Optional VirtChnl-version-gated features. VF sends rule install request; PF validates + installs HW rule (or rejects). Rule count quota enforced.

## Hardening

- VirtChnl msg size validated per-opcode.
- PF-response payload validated against schema before consumption.
- Reset retry bounded.
- Async event payload validated.
- Ring sizes from PF caps (not user-specified) at init.
- Fdir rule count enforced against quota.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — ethtool ioctl ingress per-opcode size validation.
- **PAX_KERNEXEC** — netdev_ops, ethtool_ops, VirtChnl message handlers, async-event dispatch, AdminQ ops, XDP ops all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — ethtool / netlink / sysfs entry paths per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `iavf_adapter` refcount, per-q_vector refcount, per-ring refcount, in-flight VirtChnl msg refcount saturating.
- **PAX_MEMORY_SANITIZE** — VirtChnl msg buffers cleared after send. Per-VF context cleared on remove.
- **PAX_UDEREF** — SMAP/SMEP on every user-facing entry.
- **PAX_RAP / kCFI** — netdev_ops, ethtool_ops, VirtChnl handler tables, async-event dispatch, XDP ops all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/iavf/<bdf>/*`) reveals ring + mailbox addresses; scrubbed for non-root.
- **GRKERNSEC_DMESG** — iavf VirtChnl trace restricted from non-root.
- **CAP_NET_ADMIN required** — every state-changing op (MAC change request, VLAN add request, RSS config, Fdir rule install). MAC change still may be refused by PF based on trust.
- **PF-as-adversary defensive validation** — every PF response + async event payload validated against schema. Closes CVE-2024-27059-class (PF response OOB read).
- **VirtChnl opcode allowlist** — only known opcodes accepted; unknown drop. Closes "compromised PF sends new opcode" class.
- **VirtChnl response size strict** — per-opcode size checked. Closes parser-OOB class.
- **Async event payload validated** — every async event payload size + type checked.
- **VirtChnl version pinned at negotiation** — runtime version change refused; closes downgrade-attack class.
- **VF resource cap enforced** — VF refuses to use more queues/MSI-X/RSS slots than PF assigned (defense in depth — PF should also enforce).
- **Reset rate-limit** — PF-initiated reset rate-limited; >3 per minute → driver-failed (closes "PF gaslights VF into reset loop" class).
- **PTP CAP_SYS_TIME** — clock-adjust requires CAP_SYS_TIME inside the guest.
- **Self-diag CAP_SYS_ADMIN** — gated.
- **No NVM/FW access** — VF has no hardware NVM access; refused at driver level (not relying on FW to refuse).
- **No direct register debug** — debugfs register dumps not allowed for VF (no privileged HW access).
- **Live-migration state validation** — on post-migration init, VF caps re-validated; if PF-reported caps differ from previously-known, log warning + re-init from caps.
- **Fdir rule install per-VF cap** — enforced both at VF-side request + at PF-side install. Closes a path where VF could exhaust PF Fdir table.

Per-doc rationale: iavf is unique among netdev drivers because its trust model is *inverted*: the PF (host) is the adversary, not the network. A buggy or compromised host can send malformed VirtChnl messages to the guest VF; if the VF doesn't defensively validate, the guest is compromised. CVE-2024-27059-class bugs are this exact pattern. The Rust translation's typed VirtChnl response parsing + per-opcode size validation + opcode allowlist close the class structurally. Reset rate-limit closes the "PF gaslights VF into reset loop" denial-of-service. Resource cap enforced VF-side (defense in depth) closes the "compromised PF over-grants resources to trick VF into invalid state" class.

## Open Questions

- [ ] Q1: VirtChnl version compatibility matrix — how many old versions to support? iavf historically goes back to VC v1.0 (XL710 first-gen). Recommendation: support all VC versions in upstream baseline; deprecation only via explicit removal.
- [ ] Q2: Live migration testing — needs full KVM live-migration test harness; how to integrate into Rookery's verification?
- [ ] Q3: iavf-on-baremetal — some deployments expose iavf to the host directly (passthru via vfio-pci to the host OS itself). Edge case; defensive validation still applies.
- [ ] Q4: PF-trust mode — when running on a trusted PF (e.g. on-prem datacenter where host = tenant), some validation could be relaxed for perf. Recommendation: no — the cost is small + the safety guarantee is essential.

## Verification

- **Kani SAFETY**: prove every VirtChnl response handler validates size before dereferencing payload fields.
- **TLA+**: model the reset state machine + concurrent VirtChnl async events; check converges to consistent state.
- **Verus**: functional spec of GET_VF_RESOURCES response parser — for valid response, extracts caps; for malformed, returns -EINVAL.
- **Kani+Verus**: invariant that every VirtChnl request has a tracked response slot; cleanup at adapter remove releases all.
- **Integration**: KVM guest with VF passthrough — iperf3 line rate, VLAN ops, MAC change (trust-permitting), reset stress, live-migrate stress. Cross-PF compat: same iavf binary against XL710 + E810 PFs.
- **Fuzz**: VirtChnl response fuzzer (mutate every response payload — VF must reject without panic). Async event fuzzer.
- **Penetration**: from the host side (simulated compromised PF), send malformed VirtChnl messages; iavf in guest must not panic.

## Out of Scope

- i40e PF — covered in `i40e.md`
- ice PF — covered in pre-existing `intel-ice.md`
- VirtChnl ABI doc — shared crate, separate Tier-3
- Other Intel VF drivers (i40evf, legacy) — covered by iavf via VirtChnl version
- KVM live-migration internals — out of scope
- vfio-pci passthrough mechanics — covered in `drivers/vfio/`
