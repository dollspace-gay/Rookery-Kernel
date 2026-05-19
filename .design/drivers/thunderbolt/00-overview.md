# Tier-3: drivers/thunderbolt/ — USB4 / Thunderbolt 3 stack (ctl + icm + nhi, tunnels, host/device router)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/thunderbolt/
  - include/linux/thunderbolt.h
-->

## Summary

The Thunderbolt subsystem implements the Linux side of the USB4 / Thunderbolt 3 stack — a protocol multiplexer that tunnels PCIe, DisplayPort, USB 3.x, and host-to-host DMA over a single bidirectional differential link at up to 40 Gb/s (TB3) or 80 Gb/s (USB4 v2 asymmetric). Each link end is a *router*: the *host router* lives in the SoC/PCH and exposes itself to Linux via the NHI (Native Host Interface) PCIe function; *device routers* live in docks, eGPUs, and downstream peripherals. Routers chain into a tree rooted at the host; the Connection Manager (Linux side) owns tree topology + tunnel allocation. Two CM flavors exist: ICM (firmware-resident, used on older Apple + some TB3 platforms) and native Linux CM (used on USB4 + modern TB3 with `ThunderboltSoftwareConnectionManager` ACPI).

This Tier-3 overview spans the full subsystem (~30 files, ~25k LOC): the CM core (`tb.c`, `tb.h`), control channel (`ctl.c`, `ctl.h`), router switch model (`switch.c`), tunnel mgmt (`tunnel.c`, `tunnel.h`), bus/domain layer (`domain.c`), NHI PCIe HW (`nhi.c`, `nhi.h`, `nhi_regs.h`, `nhi_ops.c`), ICM (`icm.c`), USB4 router ops (`usb4.c`, `usb4_port.c`), per-device features (`cap.c`, `tmu.c`, `lc.c`, `clx.c`, `retimer.c`), NVM (`nvm.c`), DMA test (`dma_test.c`), and XDomain (`xdomain.c`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tb` | per-domain (one per NHI) state: security level, lock, root switch, CM ops | `thunderbolt::Domain` |
| `struct tb_cm_ops` | CM vtable: `start`, `stop`, `suspend`, `complete`, `approve_switch`, `add_switch_key`, `challenge_switch_key`, `disconnect_pcie_paths`, `approve_xdomain_paths`, `disconnect_xdomain_paths` | `thunderbolt::CmOps` |
| `struct tb_switch` | per-router state: parent, depth, uuid, vendor/device, security_level, authorized, key, ports[] | `thunderbolt::Switch` |
| `struct tb_port` | per-port state: type, hopids, link, in/out adapters | `thunderbolt::Port` |
| `struct tb_tunnel` | per-tunnel state: type (PCI/DP/USB3/DMA), src/dst ports, hop chain, paths[] | `thunderbolt::Tunnel` |
| `struct tb_nhi` | per-NHI PCIe-function HW: iobase, hop_count, tx/rx rings, MSI-X vectors | `thunderbolt::Nhi` |
| `struct tb_ctl` | per-domain control channel: TX/RX rings, request queue, frame DMA pool | `thunderbolt::Ctl` |
| `struct tb_xdomain` | host-to-host peer router state | `thunderbolt::XDomain` |
| `struct tb_service` | per-protocol-handler service (network, KVMs, etc.) | `thunderbolt::Service` |
| `tb_domain_alloc(nhi, timeout_msec, privsize)` / `tb_domain_add(tb, reset)` / `tb_domain_remove(tb)` | per-NHI domain lifecycle | `Domain::alloc` / `_add` / `_remove` |
| `tb_domain_approve_switch(tb, sw)` / `_approve_switch_key(...)` / `_challenge_switch_key(...)` | security-level enforcement | `Domain::approve_switch_*` |
| `tb_switch_alloc(tb, parent, route)` / `tb_switch_add(sw)` / `tb_switch_remove(sw)` | per-router lifecycle | `Switch::alloc` / `_add` / `_remove` |
| `tb_tunnel_alloc_pci(tb, up, down)` / `_alloc_dp(...)` / `_alloc_usb3(...)` / `_alloc_dma(...)` | per-protocol tunnel alloc | `Tunnel::alloc_*` |
| `tb_tunnel_activate(tunnel)` / `_deactivate(tunnel)` | per-tunnel enable/disable | `Tunnel::activate` / `_deactivate` |
| `tb_ctl_alloc(nhi, ...)` / `tb_ctl_start(ctl)` / `tb_ctl_stop(ctl)` | per-domain control channel | `Ctl::alloc` / `_start` / `_stop` |
| `tb_cfg_read(ctl, ...)` / `tb_cfg_write(ctl, ...)` / `tb_cfg_get_upstream_port(...)` | control-packet config-space access | `Ctl::cfg_read` / `_write` |
| `nhi_probe(pdev, id)` / `nhi_remove(pdev)` | NHI PCI driver bind/unbind | `Nhi::probe` / `_remove` |

## Compatibility contract

REQ-1: Per-NHI PCI function probed via `nhi_probe`; each NHI gets exactly one `struct tb_domain`. NHI hardware is identified by PCI vendor/device ID (Intel Alpine Ridge, Titan Ridge, Maple Ridge, Barlow Ridge; AMD Yellow Carp; vendor docks).

REQ-2: USB4 spec compliance: domain announces USB4 v1 or v2 capability via NHI ECAP; v2 supports asymmetric link (80G TX / 20G RX or vice versa), enhanced DPRx, and CL-state TMU sync.

REQ-3: Security levels (per `tb_security_names`): `none`, `user`, `secure`, `dponly`, `usbonly`, `nopcie`. Default user-mode requires explicit `authorized=1` write via sysfs before PCIe tunnel is established. `secure` adds challenge-response with stored switch key.

REQ-4: Router tree: host router at depth 0; downstream switches at depth 1..6 (max TB3) / 1..N (USB4 supports deeper). Route string is a 64-bit path of per-hop port numbers.

REQ-5: Control packets carried over CTL ring (NHI hop 0): READ/WRITE/HOTPLUG/EVENT/XDOMAIN/CHANGE notifications; CRC32 protected; per-route delivery guaranteed by hardware.

REQ-6: Tunnels mapped to NHI hops 1..N (TB3: 8 hops, USB4: up to 64); each tunnel installs hop entries in every traversed router via control packets.

REQ-7: PCIe tunnel: source = host PCIe DSP, dest = downstream router PCIe USP; established via per-router PCI adapter enable + per-hop PATH config + ATS/PRI gating.

REQ-8: DP tunnel: source = GPU DP source adapter, dest = downstream DP sink; DPRX establishment timeout 12s post-tunnel-up; bandwidth allocation via DPCD requests.

REQ-9: USB3 tunnel: source = xHCI USB3 root port adapter on host router, dest = USB3 downstream adapter on device router; reserves PCIe/USB3 weight per USB4 v2 spec.

REQ-10: DMA tunnel (host-to-host XDomain): pair of routers reach each other through a peer-to-peer connection; used for `thunderbolt-net` and `thunderbolt-test`.

REQ-11: ICM-mode domains (Apple T2, some TB3 PCH): firmware-driven CM; Linux observes via NHI mailbox + ICM_GET_TOPOLOGY / ICM_APPROVE_DEVICE.

REQ-12: NVM firmware update: per-switch NVM image staged via `nvmem` API; commit requires hot-reset of the router + chain.

REQ-13: Bandwidth groups (USB4 v2): up to 7 groups (`MAX_GROUPS = 7`); DP tunnels are grouped to share bandwidth allocation across multiple monitors.

REQ-14: Asymmetric link switching: threshold `TB_ASYM_THRESHOLD = 45000 Mb/s`; above threshold, link reconfigures to asymmetric (3 TX + 1 RX lanes); below, returns to symmetric.

## Acceptance Criteria

- [ ] AC-1: Plug Intel TB3 dock into Alpine Ridge / Titan Ridge / Maple Ridge host; `boltctl list` shows dock; `bolt enroll` succeeds.
- [ ] AC-2: USB4 v2 link on Tiger Lake / Alder Lake / Meteor Lake host: 80 Gb/s symmetric link observable via `/sys/bus/thunderbolt/devices/<dom>/link_speed`.
- [ ] AC-3: PCIe tunnel: NVMe SSD in TB3 dock enumerated as PCIe device after `authorized=1`.
- [ ] AC-4: DP tunnel: 4K@60Hz monitor in dock lights up; GPU sees DP sink via i915/amdgpu.
- [ ] AC-5: USB3 tunnel: USB3 device in dock enumerated via host xHCI through TB.
- [ ] AC-6: XDomain DMA: `thunderbolt-net` brings up `tbnet0` between two Linux hosts; iperf3 achieves ≥ 20 Gb/s.
- [ ] AC-7: NVM update: `boltctl power on; boltctl forget; boltctl update-firmware` on a Lenovo dock succeeds.
- [ ] AC-8: Security level `secure`: `boltctl enroll --policy auto --key` and subsequent replug requires challenge-response.
- [ ] AC-9: ICM-mode Apple T2 host: device-add events surface via ICM mailbox; `boltctl list` reports.
- [ ] AC-10: Asymmetric link: drive DP bandwidth above 45 Gb/s, observe link auto-reconfigure to asymmetric; drop below, observe return to symmetric.

## Architecture

The subsystem layers, top to bottom:

1. **Bus / Domain** (`domain.c`): registers `thunderbolt_bus_type`; per-domain `struct tb` device under `/sys/bus/thunderbolt/devices/`; provides sysfs (`security`, `boot_acl`, `iommu_dma_protection`, switch `authorized`/`key`/`nvm_version`).
2. **CM Core** (`tb.c`): `struct tb_cm` per-domain hotplug-event queue, tunnel list, DP resource list, bandwidth groups; handles `tb_handle_hotplug` events, allocates/deallocates tunnels.
3. **Switch model** (`switch.c`): per-router `struct tb_switch` with port array, capabilities (`cap.c`), NVM (`nvm.c`), TMU (`tmu.c`), CLx (`clx.c`), LC (`lc.c`), retimer (`retimer.c`).
4. **Tunnel manager** (`tunnel.c`, `tunnel.h`): per-protocol tunnel constructors + activation; configures path table entries across the route via `tb_path_*` and `ctl.c`.
5. **Control channel** (`ctl.c`, `ctl.h`): TX/RX ring over NHI hop 0; CRC32-validated config-space and notification packets; request/response matching.
6. **NHI HW** (`nhi.c`, `nhi.h`, `nhi_regs.h`, `nhi_ops.c`): PCI driver bind; MSI-X allocation; per-ring DMA setup; mailbox; `pci_disable_link_state` for L1/L1.x as needed.
7. **ICM** (`icm.c`): alternative CM that talks to firmware via NHI mailbox; consumes `ICM_EVENT_*` and emits `ICM_APPROVE_DEVICE`.
8. **USB4 router ops** (`usb4.c`, `usb4_port.c`): USB4 spec-mandated ops not in TB3.
9. **XDomain** (`xdomain.c`): peer-to-peer DMA tunnel for `thunderbolt-net`.

Lifecycle of a hotplug:
1. NHI raises `RING_NOTIFY` IRQ; CTL ring receives `EVENT_PLUG` with route + port.
2. `tb_handle_hotplug` work queued.
3. Worker validates parent route exists, calls `tb_switch_alloc` → reads UID + caps via CTL.
4. `tb_switch_add` registers sysfs node; sets `authorized = 0` (or per security policy).
5. On userspace `echo 1 > authorized`: `tb_domain_approve_switch` builds PCIe tunnel via `tb_tunnel_alloc_pci` + `tb_tunnel_activate`.
6. DP source-side hotplug from GPU triggers DP tunnel allocation; bandwidth negotiated via group; tunnel activated.

## Hardening

(Inherits from `drivers/00-overview.md` § Hardening; thunderbolt is the highest-risk DMA-capable expansion bus.)

Thunderbolt-specific reinforcement:

- **IOMMU mandatory for external DMA** — kernel refuses to enable a PCIe tunnel unless the host PCIe RC is behind an IOMMU group with isolation; `iommu_dma_protection` sysfs attribute MUST be `1`.
- **Default `authorized=0` for new switches** — userspace explicit consent required before PCIe DMA path is opened.
- **Boot ACL on `secure` domains** — `boot_acl` is the persistent list of UUIDs that may auto-authorize at boot.
- **Per-router challenge-response** — SHA-256 over (host-nonce ‖ device-key) verifies the device is the same one previously enrolled.
- **CRC32 on every control packet** — `tb_cfg_read` / `_write` reject mismatched CRC.
- **Tunnel teardown on unplug** — `tb_handle_hotplug(unplug=true)` deactivates and frees all tunnels through the removed branch before tearing down switches.
- **Pre-boot DMA protection** — `iommu_dma_protection` reflects firmware-handoff IOMMU state (DMAR `pre_boot_protection`); if false, kernel refuses to authorize.
- **NVM update authentication** — DROM + NVM authentication via stored key + nonce; failed authentication recorded in `nvm_auth_status_cache`.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `struct tb`, `tb_switch`, `tb_port`, `tb_tunnel`, `tb_ctl`, `tb_nhi`, `tb_xdomain`, and the per-message `ctl_pkg` DMA pool; refuse direct user-buffer copies through tunnel-config paths.
- **PAX_KERNEXEC** — thunderbolt driver text (`tb.c`, `ctl.c`, `switch.c`, `tunnel.c`, `domain.c`, `nhi.c`, `icm.c`, `usb4.c`) in W^X kernel text; `tb_cm_ops`, `tb_nhi_ops`, `tb_service_driver` vtables, and the per-protocol tunnel constructors live in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `tb_handle_hotplug`, `tb_switch_add`, `tb_tunnel_activate`, `tb_ctl_rx_callback`, `nhi_interrupt_work`, and `icm_handle_event`.
- **PAX_REFCOUNT** — saturating `refcount_t` on `tb_switch`, `tb_xdomain`, `tb_service`, `tb_tunnel`, and `tb_ctl_request`; overflow trap defeats double-authorize / double-disconnect races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for switch UUID + NVM-auth keys, control-packet DMA pool buffers, tunnel hop-id tables, and the per-NHI ring frames so stale credentials and config-space data cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs `store` (`authorized`, `key`, `nvm_authenticate`, `boot_acl`, `nvm_non_active`), every nvmem read/write, and every XDomain ioctl path.
- **PAX_RAP / kCFI** — `tb_cm_ops`, `tb_nhi_ops`, `tb_service_driver`, `tb_protocol_handler->handle`, `tb_xdomain` callbacks, and the `tb_ctl` event/callback dispatch all marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-switch UID / NVM-key / route disclosure behind CAP_SYSLOG; suppress `%p` in CTL trace events and `tb_switch_dbg` paths.
- **GRKERNSEC_DMESG** — restrict TB authorize-fail, tunnel-activate-fail, CRC-fail, NVM-auth-fail, and "device unplugged" banners to CAP_SYSLOG so attackers cannot probe per-port topology via dmesg.
- **CAP_SYS_ADMIN for authorize / NVM** — sysfs `authorized`, `key`, `nvm_authenticate`, `boot_acl`, and the XDomain DMA path-approval ioctls require CAP_SYS_ADMIN; pre-boot ACL writes additionally gated by lockdown integrity mode.
- **IOMMU mandatory (Thunderclap protection)** — refuse to build a PCIe tunnel unless the host PCIe RC is behind an isolating IOMMU group; `iommu_dma_protection` must be 1 + verified at every `tb_domain_approve_switch`; defense against Thunderclap-class DMA attacks.
- **DMA-mask enforcement** — every external device DMA capability clamped to the kernel's `dma_get_required_mask`; refuse 64-bit DMA from a router that lacks IOMMU coverage.
- **NVM update lockdown** — `nvm_authenticate` rejected when `kernel_is_locked_down(LOCKDOWN_PCI_ACCESS)`; firmware update bypasses are blocked under integrity-mode lockdown.
- **`boot_acl` size capped** — `tb->nboot_acl` (per ICM ACL size) bounded; sysfs writes that would exceed cap refused.
- **CRC32 + length validation on every CTL packet** — control-packet handler refuses oversize / truncated frames; defense against malformed packet exploiting parsers.

Rationale: Thunderbolt is a privileged DMA-capable external expansion bus — a malicious dock can in principle read host RAM at 40 Gb/s. The historical Thunderclap class of attacks demonstrated that "IOMMU enabled" is necessary but not sufficient: pre-boot windows and lax authorization let attackers in. CAP_SYS_ADMIN on `authorize`, lockdown-mode gating on NVM update, IOMMU + DMA-mask enforcement at every approve, refcount saturation on switch/tunnel, and kCFI on the CM/NHI/ICM ops collapse the attack surface from "physical access → root" to a verified consent + isolation chain.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-router NVM image format internals: future `thunderbolt/nvm.md` Tier-3.
- TMU (Time Management Unit) / CLx (Compliance) deep-dive: future `thunderbolt/tmu-clx.md`.
- ICM mailbox protocol details: future `thunderbolt/icm.md`.
- Retimer subsystem: future `thunderbolt/retimer.md`.
- DMA-test ioctl interface (`dma_test.c`): future `thunderbolt/dma-test.md`.
- thunderbolt-net (in `drivers/net/thunderbolt/`): tracked separately.
- Implementation code.
