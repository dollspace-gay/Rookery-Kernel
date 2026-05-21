# Tier-3: drivers/net/ethernet/intel/igc/{igc_main,igc_base,igc_i225,igc_mac,igc_phy,igc_nvm,igc_ethtool,igc_ptp,igc_tsn,igc_xdp,igc_diag,igc_dump,igc_leds}.c — Intel I225/I226 Foxville 2.5 Gbps Ethernet (every modern enthusiast + SMB + workstation motherboard, every NUC, every TSN-aware embedded gateway)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/net/ethernet/00-overview.md
upstream-paths:
  - drivers/net/ethernet/intel/igc/igc_main.c                  (~7870 lines: netdev ops, NAPI, RX/TX, IRQ, link, reset, watchdog)
  - drivers/net/ethernet/intel/igc/igc_base.c                  (base device ops shared between I225 and I226)
  - drivers/net/ethernet/intel/igc/igc_i225.c                  (I225-specific quirks; LTR + EEE workarounds)
  - drivers/net/ethernet/intel/igc/igc_mac.c                   (MAC ops — link, RAR, multicast, statistics)
  - drivers/net/ethernet/intel/igc/igc_phy.c                   (PHY ops — auto-neg, 1G/2.5G negotiation, EEE, downshift)
  - drivers/net/ethernet/intel/igc/igc_nvm.c                   (NVRAM/EEPROM via flash interface)
  - drivers/net/ethernet/intel/igc/igc_ethtool.c               (ethtool ops — full surface, no SR-IOV)
  - drivers/net/ethernet/intel/igc/igc_ptp.c                   (PHC + per-packet TX/RX timestamping — strong PTP-grade hardware)
  - drivers/net/ethernet/intel/igc/igc_tsn.c                   (TSN — Time-Sensitive Networking: IEEE 802.1Qbv TAS + 802.1Qav CBS + LaunchTime)
  - drivers/net/ethernet/intel/igc/igc_xdp.c                   (XDP_DRV + XDP_REDIRECT + AF_XDP zerocopy)
  - drivers/net/ethernet/intel/igc/igc_diag.c                  (HW self-diagnostic via ethtool selftest)
  - drivers/net/ethernet/intel/igc/igc_dump.c                  (debug dumps)
  - drivers/net/ethernet/intel/igc/igc_leds.c                  (LED control via Linux LED class)
  - drivers/net/ethernet/intel/igc/igc.h
-->

## Summary

igc is the Intel I225/I226 (codename Foxville) 2.5 Gigabit Ethernet driver. Foxville is the silicon that brought 2.5G/MultiGig Ethernet from the niche enterprise switch market into mainstream desktops + laptops + SMB servers + embedded gateways. Every modern motherboard from ASUS/MSI/Gigabyte/ASRock with a 2.5G LAN port ships either I225 (first-gen, 2019, had errata) or I226 (second-gen, 2021, mainstream now). Intel NUC, Beelink/Minisforum mini-PCs, OpenWrt-class routers (Synology RT, Asus RT-AX), TrueNAS/Unraid home/SMB NAS systems — all use igc. The 2.5G ecosystem grew around this silicon.

igc also has **strong TSN (Time-Sensitive Networking) support** — IEEE 802.1Qbv (Time-Aware Shaper), IEEE 802.1Qav (Credit-Based Shaper), and LaunchTime (per-frame absolute TX timing). This makes I225/I226 a popular silicon choice for industrial automation, automotive (TSN AVB), and audio-video bridging applications. Combined with strong PTP support (sub-100ns sync), igc is the entry-level TSN NIC for the open-source world.

Source: ~16K lines / 13 .c files. No SR-IOV (consumer/SMB silicon).

This Tier-3 covers all of igc.

## Rust translation posture

igc is a simpler driver than i40e/ice — no SR-IOV, no AdminQ (uses direct register access), no DCB, no FCoE, no IPsec. The TSN + PTP surface is the distinguishing complexity. Translation:

- **`igc-core` crate** — netdev, NAPI, RX/TX, link mgmt, ethtool, PHC + TSN.
- **Per-silicon `IgcSilicon` trait** — I225 vs I226 (vs future I226-V variants). Per-silicon erratas + quirks.
- **`IgcAdapter<Phase>` typed lifecycle.**
- **TSN qdisc integration** typed: `Tas { gate_list, base_time, cycle_time }` (Time-Aware Shaper), `Cbs { idleslope, sendslope, hicredit, locredit }` (Credit-Based Shaper). Userspace `tc qdisc add taprio / cbs` lowers to these.
- **LaunchTime descriptors** typed as a TX descriptor flavor. Per-TX-ring LaunchTime-capable flag.
- **PHC + PTP-grade hardware timestamping** typed `Phc` with per-packet TX/RX timestamp fields.
- **AF_XDP** standard pattern.

Grsec is mandatory. igc is less CVE-prone than the SR-IOV-bearing PFs but the TSN/PTP surface is unique attack surface (per-frame absolute time + global clock sync).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct igc_adapter` | per-netdev priv | `IgcAdapter<Phase>` |
| `struct igc_hw` | HW abstraction | `IgcHw<S>` |
| `struct igc_ring` | per-direction ring | `Ring<Direction>` |
| `struct igc_q_vector` | per-MSI-X NAPI | `QVector` |
| `igc_probe(pdev, ent)` / `_remove(pdev)` | PCI probe/remove | `IgcAdapter::probe` / `Drop` |
| `igc_open(netdev)` / `_close(netdev)` | open/close | `IgcAdapter::open` / `close` |
| `igc_xmit_frame(skb, netdev)` | TX | `IgcAdapter::xmit` |
| `igc_poll(napi, budget)` | NAPI poll | `Napi::poll` |
| `igc_msix_other(irq, data)` / `_msix_ring(irq, data)` | MSI-X | `Irq::*` |
| `igc_init_hw_base(...)` / `_init_phy_workarounds(...)` | per-silicon init | `Silicon::init_hw` |
| `igc_check_for_link_base(hw)` | link state | `Link::check` |
| `igc_setup_link_*` | auto-neg | `Link::setup` |
| `igc_setup_copper_link(...)` | 1G/2.5G copper neg | `Link::setup_copper` |
| `igc_set_rx_mode(netdev)` / `_set_mac(netdev, p)` | filter mgmt | `Filter::*` |
| `igc_ptp_init(adapter)` / `_release(...)` / `_tx_hwtstamp_work(work)` / `_rx_pktstamp(...)` | PTP | `Ptp::*` |
| `igc_tsn_offload_apply(adapter)` / `_taprio_offload(...)` / `_cbs_offload(...)` | TSN | `Tsn::*` |
| `igc_set_taprio(adapter, qopt)` / `_set_cbs(adapter, qopt)` | tc qdisc handlers | `Tsn::set_taprio` / `set_cbs` |
| `igc_xdp(netdev, xdp)` / `_xdp_xmit(...)` | XDP | `Xdp::*` |
| `igc_xsk_pool_setup(adapter, pool, qid)` / `_xsk_wakeup(...)` | AF_XDP | `Xsk::*` |
| `igc_get_drvinfo(...)` / `_get_ringparam(...)` / `_get_coalesce(...)` / `_get_ts_info(...)` | ethtool | `Ethtool::*` |
| `igc_nvm_*` | NVRAM read/write | `Nvm::*` |
| `igc_diag_*` | self-test | `Diag::*` |
| `igc_leds_*` | LED control via Linux LED class | `Leds::*` |
| `igc_aer_*` | AER handlers | `Aer::*` |

## Compatibility contract

REQ-1: PCI ID table — I225-LM, I225-V, I225-IT, I225-K2, I226-LM, I226-V, I226-IT, I226-K. Frozen.

REQ-2: Per-silicon ops — I225 has known errata (LTR, EEE — fixed via firmware errata workarounds in driver); I226 is the stable revision. Per-silicon init applies the right workaround set.

REQ-3: 2.5G / 1G / 100M / 10M auto-negotiation; downshift on failure to lower speed.

REQ-4: PHC strong timestamping — sub-100ns sync precision; PHC clock per silicon spec.

REQ-5: TSN ABI — IEEE 802.1Qbv TAS (Time-Aware Shaper) configurable via tc-taprio offload; IEEE 802.1Qav CBS (Credit-Based Shaper) via tc-cbs offload. LaunchTime (per-frame absolute TX time) via XDP_TX or per-packet flag.

REQ-6: ethtool full surface (no SR-IOV ops; no FCoE; no DCB).

REQ-7: XDP_DRV native + XDP_REDIRECT + AF_XDP zerocopy.

REQ-8: AER — pci_error_handlers.

REQ-9: NVRAM via flash interface (no AdminQ — simpler register-access).

REQ-10: LED — Linux LED class integration; per-port LED control.

REQ-11: Wake-on-LAN — magic packet, unicast filter, link-change events.

REQ-12: EEE — Energy-Efficient Ethernet support per silicon spec.

## Acceptance Criteria

- [ ] AC-1: I226-V on a modern motherboard probes; auto-negotiates 2.5G with a peer; iperf3 saturates 2.5G.
- [ ] AC-2: 2.5G with shorter-than-spec cable: downshifts to 1G; link comes up; no NEPHOG (Need Ethernet to Power-cycle-the-Host-to-Get-out-of-link-down) issues.
- [ ] AC-3: PTP: ptp4l syncs to NIC PHC within 100ns offset; verified with PHC2SYS to PHC.
- [ ] AC-4: TSN smoke — TAS: configure tc-taprio with 8 gate windows on a 1ms cycle; verify TX timing per gate via tcpdump+hwtstamp on receiver.
- [ ] AC-5: TSN smoke — CBS: configure tc-cbs with 100 Mbps idle-slope on a traffic-class; verify bandwidth shaping.
- [ ] AC-6: LaunchTime: send a frame with launch-time = T; observed TX time within ±N µs of T (per silicon spec).
- [ ] AC-7: XDP_DRV: xdp_pass + xdp_drop + xdp_redirect smoke tests.
- [ ] AC-8: AF_XDP zerocopy: xdpsock benchmark on 2.5G saturates queue.
- [ ] AC-9: Wake-on-LAN: ethtool -s eth0 wol g; suspend; magic packet wakes.
- [ ] AC-10: AER injection: cleanly recovers.

## Architecture

**Standard netdev pattern.** Per-CPU NAPI + per-CPU RX/TX rings + MSI-X. Smaller queue count than enterprise NICs (typically 4 queues per port). Page-pool RX.

**PTP-grade timestamping.** I225/I226 has on-die crystal-oscillator-disciplined PHC clock; sub-100ns sync achievable with PTP4L. Per-packet TX/RX timestamps in dedicated descriptor fields.

**TSN engine.** HW gate list (TAS): up to 8 gates with per-gate enable bitmap + duration; cycle-time + base-time programmable. Driver translates tc-taprio qopt to gate-list config. CBS engine has 4 traffic classes with per-class idle-slope/send-slope; tc-cbs handler maps qopt to CBS regs.

**LaunchTime.** TX descriptor flag + per-descriptor 32-bit launch-time. Driver enables on per-ring basis. Used for deterministic-latency embedded applications.

**Per-silicon workarounds.** I225 had multiple errata at launch (LTR misconfig causing PCIe hangs, EEE-related link flap). Driver detects silicon ID + applies the right workaround set. I226 is the stable revision with most errata fixed in silicon.

**LEDs.** Per-port LED control via Linux LED class. Used by motherboard front-panel LEDs.

**Power mgmt.** D0/D3hot/D3cold; runtime PM via PCI; EEE during link idle.

## Hardening

- Per-silicon errata workarounds applied at probe.
- PHC adjust validates input range.
- TSN configuration validated against silicon caps (gate count, cycle-time range).
- LaunchTime values bounded against PHC time + max-future-offset.
- PHY MDIO access scoped.
- AER recovery rate-limited.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — ethtool, tc qdisc, PTP ioctl ingress per-opcode size validation.
- **PAX_KERNEXEC** — netdev_ops, ethtool_ops, per-silicon ops, TSN qdisc handlers, PTP ops, XDP ops, AER handlers, LED ops all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — ethtool / tc / PTP ioctl / sysfs entry paths per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `igc_adapter` refcount, per-q_vector refcount, per-ring refcount saturating.
- **PAX_MEMORY_SANITIZE** — TSN gate-list scratch cleared on reconfig. PTP per-packet timestamp buffer cleared on free.
- **PAX_UDEREF** — SMAP/SMEP on every user-facing entry.
- **PAX_RAP / kCFI** — netdev_ops, ethtool_ops, per-silicon op tables, TSN tc-offload handlers, PTP ops, AER handlers all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs scrubbed; reveals ring + PHC pointers.
- **GRKERNSEC_DMESG** — igc verbose logs (TSN gate list, LaunchTime values, per-packet timestamps revealing traffic timing) restricted from non-root — TSN/PTP info-leak via timing patterns is a side-channel.
- **CAP_NET_ADMIN required** — every state-changing op: ethtool MAC/RSS/coalesce/ring, tc-taprio install, tc-cbs install, LaunchTime enable, WOL setup.
- **CAP_SYS_TIME for PHC adjust** — clock-adjust operations require CAP_SYS_TIME, not CAP_NET_ADMIN. Closes a path where a CAP_NET_ADMIN-only process could disrupt system time sync.
- **TSN qopt validated against silicon caps** — gate count, cycle-time, base-time all validated; out-of-range refused. Closes "user installs garbage TAS config that hangs FW" class.
- **LaunchTime range strict** — launch-time validated against `now + max_future_offset`; past or far-future times refused. Closes a path where a malicious app could fill the TX queue with frames scheduled for the distant future, blocking other traffic.
- **PHC adjust rate-limit** — frequency adjustment rate-limited to prevent oscillation attacks.
- **AER recovery rate-limit** — 3 in 60s.
- **NVRAM access CAP_SYS_ADMIN** — destructive NVRAM writes (MAC change, EEPROM data) require CAP_SYS_ADMIN.
- **Debugfs CAP_SYS_ADMIN** — writes; reads scrub addresses.
- **WOL config CAP_NET_ADMIN** — closes "unprivileged user enables WOL to prevent normal shutdown" annoyance.
- **LED config CAP_SYS_ADMIN** — closes "unprivileged user manipulates motherboard LEDs as covert channel" class.
- **AF_XDP UMEM mmu-notifier strict** — same as other Intel NICs.
- **Per-silicon errata applied unconditionally** — driver refuses to skip errata workarounds even via debug knobs (prevents an exploitable code-path via I225 known-bad mode).
- **PHY-register write allowlist** — direct PHY register write via ethtool restricted to a documented allowlist; arbitrary PHY writes refused (closes a path where a hostile PHY config could brick the link).

Per-doc rationale: igc is consumer/SMB silicon — smaller attack surface than enterprise PFs because no SR-IOV, no PF↔VF mailbox, no DCB, no FCoE, no IPsec offload. The unique attack surface is TSN + PTP: per-frame absolute time scheduling + global clock sync are the headline features and the only realistic abuse paths (TSN config DoS by filling gate list with bad config, LaunchTime DoS by scheduling distant-future TX, PHC adjust as timing-side-channel). The Rust translation's typed TSN/CBS qopt + bounded LaunchTime + PHC CAP_SYS_TIME close these structurally. Side-channel concerns: per-packet timestamp data + TSN gate-list data can reveal traffic patterns to non-root observers; GRKERNSEC_DMESG + CAP_SYS_ADMIN debugfs gates close this.

## Open Questions

- [ ] Q1: I225 silicon errata maintenance — many workarounds; some may become obsolete as I225 silicon ages out. Recommendation: keep all workarounds + add deprecation warning per-revision when production volume drops below threshold.
- [ ] Q2: TSN qdisc tc-offload — interaction with Linux net_sched / qdisc framework. Edge cases at qdisc rebind.
- [ ] Q3: LaunchTime + AF_XDP — both want per-packet TX timing control. Document the interaction.
- [ ] Q4: I226-V vs I226-LM vs I226-IT — minor differences (industrial-temp variant); preserved as per-PCI-ID quirks.

## Verification

- **Kani SAFETY**: prove TSN qopt validation cannot be bypassed; prove LaunchTime range bounded.
- **TLA+**: model TSN gate-list reconfig + concurrent TX submission; check no in-flight TX is dropped if gate-list change is timed correctly.
- **Verus**: functional spec of tc-taprio qopt → TAS gate-list lowering; for valid input, produces correct HW config.
- **Kani+Verus**: invariant that every PHC time-sample is bounded by silicon clock resolution.
- **Integration**: igc on a motherboard NIC — 2.5G sustained iperf3, PTP sync (1h soak with ptp4l + phc2sys), TSN smoke (TAS gate + CBS shaping verified with Aurora-style end-to-end timing), XDP smoke, AF_XDP throughput, WOL cycle, AER recovery.
- **Fuzz**: tc-taprio qopt fuzzer; tc-cbs qopt fuzzer; LaunchTime descriptor fuzzer; PTP ioctl fuzzer.
- **Penetration**: unprivileged container attempts TSN config — refused. PHC adjust without CAP_SYS_TIME — refused. LED write without CAP_SYS_ADMIN — refused.

## Out of Scope

- igb (legacy Intel 1G driver, sibling) — separate Tier-3
- e1000e (Intel 1G older sibling) — separate Tier-3
- ixgbe / i40e / ice — covered in their own docs
- TSN framework details (tc-taprio + tc-cbs core) — net/sched Tier-3
- Linux LED class — separate driver subsystem doc
- PTP framework — covered in drivers/ptp/ Tier-3 (pre-existing)
