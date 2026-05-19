# Tier-3: drivers/thunderbolt/{domain.c,nhi.c} — TB domain (bus + sysfs) + NHI PCIe HW

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/thunderbolt/00-overview.md
upstream-paths:
  - drivers/thunderbolt/domain.c
  - drivers/thunderbolt/nhi.c
  - drivers/thunderbolt/nhi.h
  - drivers/thunderbolt/nhi_regs.h
  - drivers/thunderbolt/nhi_ops.c
-->

## Summary

`domain.c` is the bus + sysfs layer of the Thunderbolt subsystem: it registers `thunderbolt_bus_type`, defines `struct tb` (per-NHI domain) as a `struct device` under that bus, exposes the security-level + boot-ACL + IOMMU-protection + per-switch sysfs interface, and coordinates the security-level handshake (`tb_domain_approve_switch`, `tb_domain_approve_switch_key`, `tb_domain_challenge_switch_key`). It also drives the user-visible authorization flow: `echo 1 > /sys/bus/thunderbolt/devices/<sw>/authorized` calls `tb_domain_approve_switch` which invokes `tb_cm_ops->approve_switch` (CM-side glue in `tb.c`).

`nhi.c` is the PCIe HW driver for the Native Host Interface — the PCIe function that exposes the host router to Linux. It allocates MSI-X vectors, sets up TX/RX rings + interrupt registers, drives the NHI mailbox for ICM communication, handles ring interrupts, manages `pci_disable_link_state(L1)` for power management, and provides the `tb_ring_*` API consumed by `ctl.c` (control channel) and tunnel-DMA paths. NHI hardware: Intel Alpine/Titan/Maple/Barlow Ridge, AMD Yellow Carp, future USB4 routers.

This Tier-3 covers `domain.c` (~886 lines) + `nhi.c` (~1585 lines) + `nhi.h` + `nhi_regs.h` + `nhi_ops.c`. The CM core + control channel + tunnel manager live in the sibling `core.md`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `static const struct bus_type tb_bus_type` (via `bus_register`) | thunderbolt bus | `thunderbolt::Bus` |
| `tb_service_type` (device_type) | per-protocol-handler bus device | `thunderbolt::ServiceType` |
| `tb_domain_alloc(nhi, timeout_msec, privsize)` | allocate `struct tb` for an NHI | `Domain::alloc` |
| `tb_domain_add(tb, reset)` / `tb_domain_remove(tb)` | per-domain register/unregister | `Domain::add` / `_remove` |
| `tb_domain_suspend_noirq(tb)` / `_resume_noirq(tb)` / `_suspend(tb)` / `_complete(tb)` | PM hooks | `Domain::suspend_*` / `_resume_*` |
| `tb_domain_approve_switch(tb, sw)` | level 1 (user) authorize | `Domain::approve_switch` |
| `tb_domain_approve_switch_key(tb, sw)` | level 2 (secure) authorize + store key | `Domain::approve_switch_key` |
| `tb_domain_challenge_switch_key(tb, sw)` | level 2 challenge-response (boot ACL replay) | `Domain::challenge_switch_key` |
| `tb_domain_disconnect_pcie_paths(tb)` | tear all PCIe tunnels (`secure` policy or unbind) | `Domain::disconnect_pcie_paths` |
| `tb_domain_approve_xdomain_paths(tb, xd, ...)` / `_disconnect_xdomain_paths(...)` | XDomain authorization | `Domain::approve_xdomain_paths` |
| sysfs attrs: `security`, `boot_acl`, `iommu_dma_protection`, `deauthorization` | per-domain sysfs | `Domain::sysfs` |
| `struct tb_nhi` | per-NHI HW: pdev, iobase, hop_count, lock, tx/rx rings | `thunderbolt::Nhi` |
| `nhi_probe(pdev, id)` / `nhi_remove(pdev)` / `nhi_shutdown(pdev)` | PCI bind/unbind/shutdown | `Nhi::probe` / `_remove` / `_shutdown` |
| `nhi_suspend_noirq(dev)` / `_resume_noirq(dev)` / `_runtime_suspend(dev)` / `_runtime_resume(dev)` | PM hooks | `Nhi::suspend_*` / `_resume_*` |
| `tb_nhi_msi(nhi)` / `nhi_enable_int_throttling(nhi)` | MSI-X + interrupt throttling | `Nhi::msi` / `_int_throttling` |
| `tb_ring_alloc_tx(nhi, hop, size, flags)` / `_alloc_rx(...)` / `tb_ring_start(ring)` / `tb_ring_stop(ring)` / `tb_ring_free(ring)` | per-ring lifecycle | `Ring::alloc_tx` / `_alloc_rx` / `_start` / `_stop` / `_free` |
| `__tb_ring_enqueue(ring, frame)` | enqueue DMA frame | `Ring::enqueue` |
| `nhi_mailbox_cmd(nhi, cmd, arg)` / `_mailbox_mode(nhi)` | ICM mailbox | `Nhi::mailbox_cmd` / `_mailbox_mode` |
| `ring_interrupt_active(ring, active)` / `nhi_msix(nhi)` | per-ring IRQ enable/disable | `Ring::interrupt_active` / `Nhi::msix` |
| `struct tb_nhi_ops` (nhi_ops.c) | per-generation NHI ops: `init`, `suspend_noirq`, `resume_noirq`, `runtime_suspend`, `runtime_resume`, `shutdown` | `Nhi::Ops` |

## Compatibility contract

REQ-1: `thunderbolt_bus_type` registered at `tb_domain_init`; bus matches `tb_service` devices against `tb_service_driver`s via `tb_service_match` over `(protocol_key, protocol_id, protocol_version, protocol_revision)`.

REQ-2: Each NHI PCI function probed via `nhi_probe` creates exactly one `struct tb` allocated by `tb_domain_alloc(nhi, TB_TIMEOUT, sizeof(struct tb_cm))`; `tb_domain_add(tb, host_reset=true)` registers the domain to the bus.

REQ-3: Domain device path is `/sys/bus/thunderbolt/devices/domain<N>` where N = `ida_alloc(&tb_domain_ida)`.

REQ-4: Sysfs attrs:
- `security` — read-only string per `tb_security_names[]`: `none`/`user`/`secure`/`dponly`/`usbonly`/`nopcie`.
- `boot_acl` — RW, list of UUIDs that auto-authorize at boot (ICM-driven domains); CAP_SYS_ADMIN required to write.
- `iommu_dma_protection` — read-only `0`/`1`; `1` iff host PCIe RC behind isolating IOMMU group AND pre-boot DMA protection active.
- `deauthorization` — read-only `0`/`1`; whether the domain supports revoking authorization on a connected switch.

REQ-5: `tb_domain_approve_switch(tb, sw)`: parent switch MUST be authorized first (`parent_sw->authorized`); calls `tb->cm_ops->approve_switch(tb, sw)`; on success caller (sysfs `authorized` store) sets `sw->authorized = 1`.

REQ-6: `tb_domain_approve_switch_key(tb, sw)`: parent authorized + key present (`memcmp(sw->key, zero, KEY_SIZE) != 0`); calls `cm_ops->add_switch_key(tb, sw)` + `cm_ops->approve_switch(tb, sw)`.

REQ-7: `tb_domain_challenge_switch_key(tb, sw)`: parent authorized + key present; generates 32-byte random nonce + `crypto_shash_setkey(sw->key)` + computes HMAC-SHA256(nonce ‖ sw->uid); router does same; compare via `crypto_memneq`.

REQ-8: `tb_domain_disconnect_pcie_paths(tb)`: walks `tcm->tunnel_list` and deactivates all PCIe tunnels; called on suspend, unbind, or `nopcie` security policy.

REQ-9: NHI HW: PCI vendor 0x8086 (Intel) or 0x1022 (AMD); per-generation device IDs match `nhi_ids[]`; quirks `QUIRK_AUTO_CLEAR_INT` (Intel) or no-quirk (require manual int clear).

REQ-10: NHI hop count: TB3 Alpine Ridge: 8 hops; TB3 Titan Ridge: 8; TB3 Maple Ridge / USB4: 32 or 64 depending on generation; read from `REG_HOP_COUNT`.

REQ-11: MSI-X vectors: `MSIX_MIN_VECS = 6` (2 control + 4 DMA paths); `MSIX_MAX_VECS = 16`; allocated via `pci_alloc_irq_vectors` with `PCI_IRQ_MSIX | PCI_IRQ_AFFINITY`.

REQ-12: Per-ring DMA: TX ring + RX ring per hop pair; descriptors are 16 bytes (TB3) or 32 bytes (USB4); ring size param to `tb_ring_alloc_*`; DMA mask = 64-bit.

REQ-13: NHI mailbox: 32-bit MMIO mailbox at `REG_OUTMAIL_CMD`/`REG_INMAIL_CMD`; per-cmd timeout `NHI_MAILBOX_TIMEOUT = 500 ms`; ICM-mode firmware drives most events through here.

REQ-14: Host reset: `host_reset` module param (default true); on `nhi_probe`, USB4 host router is reset via mailbox `USB4_NHI_MAILBOX_CMD_RESET`.

REQ-15: PM: `nhi_runtime_suspend` disables PCIe L1 substates, parks rings, drops domain into D3hot; `nhi_runtime_resume` restores. Suspend/resume noirq round-trips state via `tb_domain_suspend_noirq` / `_resume_noirq`.

## Acceptance Criteria

- [ ] AC-1: `nhi_probe` on Alpine Ridge / Titan Ridge / Maple Ridge / AMD Yellow Carp: domain registered; `/sys/bus/thunderbolt/devices/domain0` exists.
- [ ] AC-2: `cat /sys/bus/thunderbolt/devices/domain0/security` returns one of `none|user|secure|dponly|usbonly|nopcie` per firmware policy.
- [ ] AC-3: `iommu_dma_protection == 1` on a system with VT-d + pre-boot DMA protection active.
- [ ] AC-4: `echo 1 > authorized` on a level-1 (user) domain succeeds; PCIe tunnel comes up.
- [ ] AC-5: Level-2 (secure) enrollment: `boltctl enroll --policy auto --key` writes key + authorizes; subsequent power cycle triggers challenge-response (`tb_domain_challenge_switch_key`).
- [ ] AC-6: MSI-X allocation succeeds with ≥ 6 vectors on Alpine Ridge.
- [ ] AC-7: Ring TX/RX over hop 0 (control channel) functional after `tb_ring_start`.
- [ ] AC-8: NHI mailbox `cmd=GET_TOPOLOGY` on ICM-mode Apple T2 returns topology blob within 500 ms.
- [ ] AC-9: Suspend-to-RAM (S3) → resume cycle preserves authorized state of enrolled switches.
- [ ] AC-10: `nhi_remove` tears down all rings + frees MSI-X without UAF on inflight CTL request.

## Architecture

### Bus + domain lifecycle (`domain.c`)

`struct tb` (per-NHI domain) carries: `dev` (under `tb_bus_type`), `nhi`, `lock` (big domain lock), `root_switch`, `cm_ops` (CM vtable set by `tb_probe` in `nhi.c`), `index`, `security_level`, `nboot_acl` (ICM ACL size; 0 if not ICM), and a trailing `privdata[]` carrying `struct tb_cm`.

`tb_domain_alloc(nhi, timeout_msec, privsize)`: `kzalloc(sizeof(struct tb) + privsize)` → `mutex_init(&tb->lock)` → `tb->index = ida_alloc(&tb_domain_ida)` → `device_initialize(&tb->dev)` with `bus = &tb_bus_type` + `dev_set_name("domain%d", tb->index)`.

`tb_domain_add(tb, reset)`: `tb_ctl_start(tb->ctl)` → `tb->cm_ops->start(tb, reset)` (CM enumerates root switch) → `device_add(&tb->dev)` (visible in sysfs) → per-switch enumeration callbacks fire.

`tb_domain_approve_switch(tb, sw)` (security level 1): refuses unless `tb->security_level ∈ {USER, SECURE}`; resolves `parent_sw = tb_to_switch(sw->dev.parent)`; refuses if `!parent_sw || !parent_sw->authorized`; delegates to `tb->cm_ops->approve_switch(tb, sw)`.

### Security-level state machine

```
security_level     | new switch behavior
-------------------|-------------------------------------------------------------
TB_SECURITY_NONE   | auto-authorize on plug; no userspace gate
TB_SECURITY_USER   | authorized=0 default; userspace must echo 1 (CAP_SYS_ADMIN)
TB_SECURITY_SECURE | authorized=0; first connect requires key install (echo 2 + key)
                   |   subsequent connects: challenge-response with stored key
TB_SECURITY_DPONLY | DP tunnels allowed; PCIe + USB3 blocked
TB_SECURITY_USBONLY| USB3 tunnels allowed; PCIe blocked
TB_SECURITY_NOPCIE | PCIe tunnels forbidden (other tunnels allowed)
```

### NHI HW (`nhi.c`)

`struct tb_nhi` carries: `lock` (per-NHI spinlock for rings + mailbox), `pdev`, `ops` (`tb_nhi_ops`), `iobase` (BAR0 MMIO), `tx_rings[hop_count]` + `rx_rings[hop_count]`, `msix_ida`, `going_away` (shutdown flag), `iommu_dma_protection`, `hop_count`, `quirks` (`QUIRK_AUTO_CLEAR_INT`, `QUIRK_E2E`).

`nhi_probe` lifecycle:
1. `pci_enable_device(pdev)` + `pcim_iomap_regions(pdev, 1<<0, "thunderbolt")`.
2. `iobase = pcim_iomap_table(pdev)[0]`.
3. Read `REG_HOP_COUNT` → `nhi->hop_count`.
4. `pci_alloc_irq_vectors(pdev, MSIX_MIN_VECS, MSIX_MAX_VECS, PCI_IRQ_MSIX | PCI_IRQ_AFFINITY)`.
5. `dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64))`.
6. `pci_disable_link_state(pdev, PCIE_LINK_STATE_L1)` (per quirk).
7. `nhi->iommu_dma_protection = pcie_iommu_dma_protected(pdev)`.
8. `nhi->ops->init(nhi)` — per-generation init (Apple T2, Intel, AMD).
9. If `host_reset`: `nhi_reset(nhi)` → mailbox reset command.
10. `tb = tb_probe(nhi)` — CM constructor (delegates to `icm.c` or native `tb.c`).
11. `tb_domain_add(tb, host_reset)`.
12. `pm_runtime_*` enable.

Ring activation (`ring_interrupt_active(ring, true)`):
- Computes per-ring interrupt bit in `REG_RING_INTERRUPT_BASE`.
- For MSI-X rings: programs `REG_DMA_MISC.INT_AUTO_CLEAR` + per-ring IVR + interrupt-vector regs.
- For legacy MSI: writes mask register.

NHI ring frame DMA: `struct ring_frame` describes a single DMA frame; consumer (`ctl.c` or tunnel-DMA test) builds the frame, calls `__tb_ring_enqueue`, NHI hardware DMA-transfers to/from device, IRQ on completion, ring's callback fires.

### Mailbox (ICM mode)

`nhi_mailbox_cmd(nhi, cmd, arg)`: under `nhi->lock` write `arg` to `REG_OUTMAIL_DATA`, write `cmd` to `REG_OUTMAIL_CMD`, poll-wait up to `NHI_MAILBOX_TIMEOUT = 500 ms` for `OP_REQUEST` to clear, read response from `REG_INMAIL_DATA`. Used by `icm.c` to communicate with firmware: get topology, approve device, challenge device, get/set boot ACL.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nhi_ring_no_oob` | OOB | per-ring `hop` index < `nhi->hop_count`; head/tail wrap correctly |
| `nhi_msix_no_collision` | UNIQUENESS | per-ring MSI-X vector unique within NHI; `msix_ida` tracks |
| `domain_lock_held_on_approve` | LOCK | `tb_domain_approve_switch` requires `tb->lock` to be held |
| `boot_acl_size_bounded` | OOB | `nboot_acl` ≤ 64 (ICM max); writes refused beyond cap |
| `mailbox_timeout_bounded` | LIVENESS | every `nhi_mailbox_cmd` returns within `NHI_MAILBOX_TIMEOUT` |

### Layer 2: TLA+

`models/thunderbolt/domain_security.tla` (parent-declared): models security-level state machine across (plug, authorize, key-install, replug, suspend, resume). Property: `TB_SECURITY_SECURE` ⇒ no PCIe tunnel exists on a switch whose `authorized != 2`.

`models/thunderbolt/nhi_ring.tla`: models concurrent ring enqueue + IRQ-side dequeue + ring_stop. Property: no in-flight frame leaks after `tb_ring_stop`; no UAF on `ring_frame` post-stop.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Domain::approve_switch` post: `parent.authorized == true ∧ cm_ops.approve_switch returned 0` | `Domain::approve_switch` |
| `Domain::challenge_switch_key` post: HMAC verified via `crypto_memneq` (constant time) | `Domain::challenge_switch_key` |
| `Nhi::probe` post: `nhi.iobase` mapped ∧ MSI-X allocated ∧ `iommu_dma_protection` set from PCIe RC | `Nhi::probe` |
| `Nhi::mailbox_cmd` post: response read iff `REG_OUTMAIL_CMD.OP_REQUEST` cleared within timeout | `Nhi::mailbox_cmd` |
| `Domain::disconnect_pcie_paths` post: no tunnel of type PCI in `tcm.tunnel_list` | `Domain::disconnect_pcie_paths` |

### Layer 4: Verus/Creusot functional

Boot to authorized PCIe tunnel: `nhi_probe` → `tb_domain_add` → CM enumerates switch → sysfs `authorized=1` write → `tb_domain_approve_switch` → CM builds PCIe tunnel → NHI ring active → device DMA-visible. Encoded as a refinement chain from `NhiHw::reset` to `Tunnel::active(PCIe)`; suspend/resume preserves the chain.

## Hardening

(Inherits from `drivers/thunderbolt/00-overview.md` § Hardening.)

`domain.c` / `nhi.c` specific reinforcement:

- **`tb->lock` per domain serializes authorize + tunnel mgmt** — defense against UAF across concurrent unplug + approve.
- **`iommu_dma_protection` sourced from PCIe RC + ACPI DMAR** — driver refuses to enable PCIe tunnels when 0 (configurable but defaults to deny).
- **`boot_acl` writes CAP_SYS_ADMIN** — `boot_acl_store` checks `capable(CAP_SYS_ADMIN)`.
- **`security_level` is read-only** — firmware-determined; no kernel API to escalate.
- **Constant-time HMAC compare** — `crypto_memneq` rather than `memcmp` defeats timing side-channels on challenge-response.
- **`host_reset` default true** — defense against firmware-side stale state on cold boot.
- **MSI-X with `IRQ_AFFINITY`** — IRQs pinned per CPU; defense against IRQ-storm overwhelming a single CPU.
- **`pci_disable_link_state(L1)` per quirk** — defense against L1.2 race with control-channel IRQ delivery.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `struct tb`, `tb_nhi`, `tb_ring`, `ring_frame`, the mailbox response buffers, and per-domain `boot_acl` arrays; refuse direct user-buffer copies into NHI MMIO mapping.
- **PAX_KERNEXEC** — `domain.c` + `nhi.c` + `nhi_ops.c` text in W^X kernel text; `tb_cm_ops`, `tb_nhi_ops`, `tb_service_driver` vtables, NHI mailbox dispatch, and the per-ring callback table `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `nhi_probe`, `nhi_interrupt`, `nhi_interrupt_work`, `tb_domain_add`, `tb_domain_approve_switch`, `tb_domain_challenge_switch_key`, and `nhi_mailbox_cmd`.
- **PAX_REFCOUNT** — saturating `refcount_t` on `tb` (via `kref_*`), `tb_ring`, `tb_xdomain`, `tb_service`, and the per-mailbox-command waiter; overflow trap defeats double-suspend / double-remove races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for the DMA-coherent NHI ring descriptors, the mailbox response buffers, `boot_acl` UUID arrays, and per-switch key material at domain teardown so credentials and topology bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs `_store` callback (`authorized`, `key`, `boot_acl`, `deauthorization`) and every XDomain ioctl path.
- **PAX_RAP / kCFI** — `tb_cm_ops`, `tb_nhi_ops`, `tb_service_driver`, NHI mailbox callback, ring `start`/`stop`/`enqueue` callbacks, and per-ring `frame_completed` dispatchers marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-NHI iobase / per-switch UID disclosure behind CAP_SYSLOG; suppress `%p` in NHI MMIO traces.
- **GRKERNSEC_DMESG** — restrict NHI probe-fail, mailbox-timeout, ring-error, authorize-fail, and "switch challenge failed" banners to CAP_SYSLOG.
- **NHI DMA-coherent ring** — TX/RX ring descriptors allocated via `dma_alloc_coherent` with explicit DMA-mask validation (64-bit on USB4, 32-bit fallback on legacy); per-frame DMA-direction enforced (`DMA_TO_DEVICE` for TX, `DMA_FROM_DEVICE` for RX); defense against ring-descriptor corruption.
- **Firmware-auth challenge-response** — `tb_domain_challenge_switch_key` uses 32-byte cryptographically-random nonce + HMAC-SHA256(key, nonce ‖ uid) + constant-time `crypto_memneq` compare; defense against replay + timing side-channels.
- **Hot-add ATS + IOMMU required** — `tb_domain_approve_switch` refuses to call `cm_ops->approve_switch` when `nhi->iommu_dma_protection == 0` OR when the host PCIe RC ATS capability is disabled; defense against Thunderclap-style hot-add bypasses.
- **CAP_SYS_ADMIN on all sysfs writes** — `boot_acl_store`, `deauthorization_store`, switch `authorized_store`, `nvm_authenticate_store` all check `capable(CAP_SYS_ADMIN)`; non-root writes refused with EPERM.
- **Mailbox timeout bounded** — `NHI_MAILBOX_TIMEOUT = 500 ms` enforced; defense against firmware hang causing infinite kernel wait.
- **`going_away` flag** — `nhi_remove` sets this before tearing down rings; ring callbacks check + bail; defense against IRQ delivery into freed memory.
- **`pci_disable_link_state(L1)` per quirk** — defense against L1.2 transitioning out from under a CTL command, dropping it.

Rationale: `domain.c` + `nhi.c` are the privileged border between the host PCIe root complex and an externally-pluggable, DMA-capable bus. A missing IOMMU check on approve lets a malicious dock walk host RAM. A weak nonce on challenge-response allows replay enrollment of an attacker's device under a stolen key. A racy `going_away` flag during NHI remove lets an in-flight DMA frame UAF the ring. CAP_SYS_ADMIN on every sysfs write, IOMMU + ATS hard-gating, constant-time HMAC, and bounded mailbox timeouts collapse the attack surface to a verified hot-plug consent boundary backed by hardware isolation.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- CM core (`tb.c`), control channel (`ctl.c`), switch model (`switch.c`), tunnel manager (`tunnel.c`): covered in `thunderbolt/core.md`.
- ICM-mode CM (`icm.c`): future `thunderbolt/icm.md` Tier-3.
- USB4 router ops (`usb4.c`, `usb4_port.c`): future `thunderbolt/usb4.md`.
- DMA test (`dma_test.c`): future `thunderbolt/dma-test.md`.
- DMA port (`dma_port.c`): future `thunderbolt/dma-port.md`.
- DROM / EEPROM (`eeprom.c`): future `thunderbolt/eeprom.md`.
- Implementation code.
