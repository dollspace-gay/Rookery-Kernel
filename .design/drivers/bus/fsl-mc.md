# Tier-3: drivers/bus/fsl-mc/ — NXP Management Complex (MC) bus / DPAA2 framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/bus/fsl-mc/fsl-mc-bus.c
  - drivers/bus/fsl-mc/dprc-driver.c
  - drivers/bus/fsl-mc/dprc.c
  - drivers/bus/fsl-mc/fsl-mc-allocator.c
  - drivers/bus/fsl-mc/fsl-mc-msi.c
  - drivers/bus/fsl-mc/fsl-mc-uapi.c
  - drivers/bus/fsl-mc/mc-io.c
  - drivers/bus/fsl-mc/mc-sys.c
  - drivers/bus/fsl-mc/obj-api.c
  - include/linux/fsl/mc.h
-->

## Summary

The Freescale / NXP Management Complex (fsl-mc) bus driver implements the Linux device-model representation of the DPAA2 (Data Path Acceleration Architecture, 2nd generation) framework used by NXP QorIQ Layerscape SoCs (LS1088, LS2080, LS2088, LS2160, etc.). The Management Complex is an on-chip firmware engine that owns the system's network and storage data-path hardware — Ethernet MACs, packet I/O, buffer pools, scheduling, classification, crypto, RTC — and exposes them to the host as MC objects (DPxxx: DPRC, DPNI, DPSW, DPIO, DPBP, DPCON, DPMCP, DPMAC, DPRTC, DPSECI, DPDMUX, DPDCEI, DPAIOP, DPCI, DPDMAI, DPDBG). The host enumerates these objects by walking the root DPRC (Data Path Resource Container) via a mailbox-style command interface (`fsl_mc_command` written into per-portal MMIO region), and each MC object becomes a `fsl_mc_device` on the `fsl_mc_bus_type` so that Linux device drivers can probe and bind to them. The bus driver also provides the MC-bus-aware MSI domain (via Layerscape ITS), per-object resource allocation, hot-add via `dprc_scan_objects`, and a `/dev/dprcN` UAPI for FreeRTOS-style userland MC commands.

This Tier-3 covers `fsl-mc-bus.c` (~1254 lines: bus_type, root probe, device add/remove, attributes), `dprc-driver.c` (~876 lines: DPRC scan + IRQ handling), `dprc.c` (~700 lines: MC command wrappers), `fsl-mc-allocator.c` (~637 lines: resource pools), `fsl-mc-msi.c` (~233 lines: MSI domain), `fsl-mc-uapi.c` (~604 lines: ioctl entry).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct fsl_mc_device` | per-MC-object Linux device (DPNI, DPSW, …) | `drivers::fsl_mc::Device` |
| `struct fsl_mc_driver` | per-driver descriptor with `match_id_table: fsl_mc_device_id[]` | `drivers::fsl_mc::Driver` |
| `struct fsl_mc_bus` (root DPRC private state) | per-DPRC bus container | `drivers::fsl_mc::DprcBus` |
| `fsl_mc_bus_type` | `bus_type` registered at module_init | `drivers::fsl_mc::BUS_TYPE` |
| `fsl_mc_bus_dprc_type` / `_dpni_type` / `_dpsw_type` / `_dpio_type` / `_dpbp_type` / `_dpcon_type` / `_dpmcp_type` / `_dpmac_type` / `_dprtc_type` / `_dpseci_type` / `_dpdmux_type` / `_dpdcei_type` / `_dpaiop_type` / `_dpci_type` / `_dpdmai_type` | per-object-class device_type | `drivers::fsl_mc::ObjType::*` |
| `fsl_mc_bus_match(dev, drv)` / `_uevent` / `_probe` / `_remove` / `_shutdown` | bus_type ops | `drivers::fsl_mc::Bus::*` |
| `__fsl_mc_driver_register(drv, owner)` / `fsl_mc_driver_unregister(drv)` | driver register UAPI | `drivers::fsl_mc::register_driver` / `_unregister` |
| `fsl_mc_device_add(...)` / `fsl_mc_device_remove(...)` | per-object lifecycle | `drivers::fsl_mc::Device::add` / `_remove` |
| `dprc_scan_objects(mc_bus_dev, alloc_interrupts)` | enumerate child objects under a DPRC | `drivers::fsl_mc::Dprc::scan_objects` |
| `dprc_scan_container(mc_bus_dev, alloc_interrupts)` | resync entire DPRC contents | `drivers::fsl_mc::Dprc::scan_container` |
| `dprc_open` / `_close` / `_get_attributes` / `_get_obj_count` / `_get_obj` / `_get_obj_region` / `_set_obj_irq` / `_reset_container` / `_get_connection` / `_get_container_id` / `_get_api_version` | MC command wrappers | `drivers::fsl_mc::Dprc::*` |
| `fsl_mc_get_version(mc_io, cmd_flags, mc_version)` | read MC firmware version | `drivers::fsl_mc::McIo::get_version` |
| `fsl_mc_object_allocate` / `_free` (in `fsl-mc-allocator.c`) | per-DPxxx resource pool alloc | `drivers::fsl_mc::Allocator::*` |
| `fsl_mc_msi_domain_alloc_irqs` / `_free_irqs` | MSI alloc against ITS parent | `drivers::fsl_mc::Msi::alloc` / `_free` |
| `fsl_mc_uapi_create_device_file` / `_remove_device_file` | `/dev/dprcN` misc-device | `drivers::fsl_mc::Uapi::create` / `_remove` |
| `fsl_mc_get_endpoint(mc_dev)` | per-object peer lookup (e.g. DPNI ↔ DPMAC) | `drivers::fsl_mc::Device::get_endpoint` |
| `mc_send_command(mc_io, &cmd)` | per-portal mailbox MMIO write + poll | `drivers::fsl_mc::McIo::send_command` |

## Compatibility contract

REQ-1: `fsl_mc_bus_type.name == "fsl-mc"`; `driver_override = true` (matches by name when set); ABI surface `/sys/bus/fsl-mc/`.

REQ-2: Root DPRC bound via `fsl_mc_bus_driver` matching `compatible = "fsl,qoriq-mc"` platform device; MMIO regions: MC-register-bank (FSL_MC_GCR1, FSL_MC_FAPR) + MC-portal-base for command channels.

REQ-3: GCR1 bits: `GCR1_P1_STOP = BIT(31)`, `GCR1_P2_STOP = BIT(30)` (per-portal halt); FAPR bits: `MC_FAPR_PL = BIT(18)`, `MC_FAPR_BMT = BIT(17)` (privilege + bypass-MMU-translation for MC initiators).

REQ-4: Per-object-class `device_type` (16 known classes); userland modalias `fsl-mc:vNNNNdNNNN` (vendor + obj-type) used by udev/modprobe.

REQ-5: DPRC = container that owns a set of objects; root DPRC owned by host; nested DPRCs supported (used by VFIO/IOMMU passthrough — DPRC = isolation domain for VFIO).

REQ-6: Match algorithm: `fsl_mc_bus_match` first honors `driver_override` (writable sysfs), else walks `match_id_table` matching `(vendor, obj_type, ver_major, ver_minor)`.

REQ-7: Hot-add: write `1` to `/sys/bus/fsl-mc/devices/dprc.N/rescan` → `dprc_scan_objects(true)` enumerates new objects + binds.

REQ-8: DMA + IOMMU: `fsl_mc_dma_configure` calls `of_dma_configure` + attaches IOMMU default domain unless driver `driver_managed_dma` flag set.

REQ-9: MSI: per-DPRC MSI domain created on top of GIC-v3 ITS parent; per-object MSI vectors allocated via `fsl_mc_msi_domain_alloc_irqs`; ICID (Isolation Context ID) maps device → SMMU stream-id.

REQ-10: UAPI `/dev/dprc.N` misc-device: `FSL_MC_SEND_MC_COMMAND` ioctl forwards arbitrary MC command from userspace to MC firmware via that DPRC's portal; gated CAP_NET_ADMIN + CAP_SYS_ADMIN.

REQ-11: Endpoint linkage (`fsl_mc_get_endpoint`): per-DPNI lookup of peer DPMAC (network port) or DPDMUX or DPNI (virtual interface) — required for tc/tcp offload.

REQ-12: Per-object IRQ: `dprc_set_obj_irq` programs the MC to route a per-object event IRQ to a host MSI vector.

## Acceptance Criteria

- [ ] AC-1: On LS1088 / LS2088 / LS2160 board: `ls /sys/bus/fsl-mc/devices/` shows root `dprc.1` plus child objects (DPNI, DPMAC, DPIO, DPBP, …).
- [ ] AC-2: `ls /sys/bus/fsl-mc/drivers/` shows fsl_dpaa2_eth, fsl_dpaa2_io, fsl_dpaa2_bp, fsl_dpaa2_sw bindings.
- [ ] AC-3: `cat /sys/bus/fsl-mc/devices/dpni.1/modalias` returns `fsl-mc:v...d...`.
- [ ] AC-4: Hot-add: create new DPNI via restool → `echo 1 > /sys/bus/fsl-mc/devices/dprc.1/rescan` → new `dpni.N` appears + dpaa2-eth binds.
- [ ] AC-5: Multi-DPRC test: create child DPRC; bind to vfio-fsl-mc → child DPRC objects exposed to userspace; root DPRC unchanged.
- [ ] AC-6: MSI test: per-DPNI interrupt fires on packet event; `cat /proc/interrupts` shows fsl-mc-msi line.
- [ ] AC-7: `/dev/dprc.1` ioctl `FSL_MC_SEND_MC_COMMAND` from privileged userland queries MC firmware version; result matches `fsl_mc_get_version`.
- [ ] AC-8: Network test: dpaa2-eth bound to DPNI + DPMAC endpoint → `ip link` shows interface; ping works.

## Architecture

`Bus::probe(dev)` on root MC platform-device:
1. ioremap MC register bank (GCR1, FAPR) + per-portal MMIO regions.
2. Read FAPR for privilege + BMT bits; verify MC firmware ready (`GCR1_P*_STOP` clear).
3. Allocate root `fsl_mc_device` representing root DPRC with `device_type = fsl_mc_bus_dprc_type`.
4. `dprc_get_container_id` → root container ID.
5. `dprc_get_attributes` → version + options.
6. `dprc_scan_objects(root, true)` walks objects under root DPRC and `fsl_mc_device_add`s each.

Per-object add `Device::add(dprc_parent, obj_desc)`:
1. Allocate `fsl_mc_device`, set `dev.bus = &fsl_mc_bus_type`, `dev.type = fsl_mc_get_device_type(obj_desc.type)`.
2. Populate `obj_desc` (vendor, type, ID, ver_major, ver_minor, region_count, irq_count, flags).
3. Per-region: ioremap MMIO regions exported by MC (per-object MMIO via `dprc_get_obj_region`).
4. Allocate ICID + MSI domain hookup.
5. `device_add` → bus_match runs against all registered fsl_mc_drivers.

DPRC scan `Dprc::scan_objects(mc_bus_dev, alloc_irqs)`:
1. `dprc_get_obj_count` → N.
2. For i in 0..N: `dprc_get_obj` → descriptor.
3. For each desc not already in bus's child list: `fsl_mc_device_add`.
4. For each existing child not in desc list: `fsl_mc_device_remove` (hot-removal).
5. If `alloc_irqs`: allocate MSI vectors per-child.

MC command path `McIo::send_command(cmd)`:
1. Per-portal MMIO write of 56-byte command header to portal base + 0.
2. Mailbox semantics: poll completion status bit in word[0]; on timeout return `-ETIMEDOUT`.
3. Read response back into `cmd` struct.

UAPI `Uapi::ioctl(FSL_MC_SEND_MC_COMMAND)`:
1. Copy command bytes from userland (PAX_USERCOPY-bounded).
2. Validate command opcode against allow-list (DPRC-scoped queries; refuse cross-DPRC ops without CAP_SYS_ADMIN).
3. Send via that DPRC's `mc_io` portal.
4. Copy response to userland.

MSI alloc `Msi::alloc(mc_dev, nvec)`:
1. Walk parent chain to find DPRC root + its MSI domain.
2. `msi_domain_alloc_irqs(parent_domain, &mc_dev->dev, nvec)`.
3. Per-vector hwirq derived from ICID.

## Hardening

- **Per-DPRC `mutex`** — concurrent scan + hot-add safe.
- **MC command timeout bounded** — refuse to spin forever on a hung MC firmware.
- **`dprc_scan_objects` removal-set computed before add-set** — orphan children removed first to prevent endpoint dangling.
- **Per-object MSI freed on device_del** — no leaked vectors.
- **`/dev/dprcN` misc-device** — per-DPRC; opening grants only that DPRC's command space.
- **Endpoint lookup `fsl_mc_get_endpoint` refuses cross-container peer** — defense against DPRC-isolation bypass.
- **Allocator pool refcounts** — per-DPIO / DPBP / DPCON / DPMCP shared across drivers, refcounted alloc.
- **Hot-remove drains in-flight commands** — `mc_io` portal quiesced before unmap.
- **`driver_override`** — bounded length via `driver_set_override`.
- **DPRC reset** — `dprc_reset_container` available but gated; reset propagates to all children.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `fsl_mc_device`, `fsl_mc_bus`, `fsl_mc_io`, `fsl_mc_command` scratch, `obj_desc`, and `dprc_attributes`; UAPI `FSL_MC_SEND_MC_COMMAND` payload bounded to 56-byte command + response.
- **PAX_KERNEXEC** — fsl-mc driver core in W^X kernel image; `fsl_mc_bus_type`, per-object `device_type` table, command opcode dispatch, and MC firmware version table placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `fsl_mc_bus_match`, `dprc_scan_objects`, `mc_send_command`, and UAPI ioctl entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on `fsl_mc_device`, `fsl_mc_bus`, and `mc_io` so concurrent hot-add/hot-remove + driver-unload cannot underflow into UAF.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `fsl_mc_device`, MC command scratch, per-DPRC bus state, MSI-vector metadata, and per-DPxxx object descriptors; DPNI / DPSW state holding network MACs and classifier rules wiped on object destroy.
- **PAX_UDEREF** — SMAP/PAN enforced on UAPI ioctl; MC commands copied via `copy_from_user` only; refuse raw user-pointer deref in command dispatch.
- **PAX_RAP / kCFI** — `fsl_mc_bus_type` ops (`match`, `uevent`, `probe`, `remove`, `shutdown`, `dma_configure`, `dma_cleanup`), per-driver `match_id_table` pointer, MSI domain ops, and DPRC IRQ-handler vtable marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of fsl-mc + DPAA2 symbols behind CAP_SYSLOG; suppress `%p` of `fsl_mc_device`, `mc_io->regs`, and `mc_bus` pointers in dev_dbg/dev_err/tracepoints.
- **GRKERNSEC_DMESG** — restrict MC firmware-error, command-timeout, DPRC-scan, and hot-add banners to CAP_SYSLOG so attackers cannot enumerate DPxxx objects via dmesg.
- **MC bus root CAP_SYS_ADMIN** — driver-override store, rescan-store, autorescan-store, and `/dev/dprcN` open require CAP_SYS_ADMIN in the owning user-ns.
- **DPRC create / destroy permission** — `dprc_reset_container` and cross-container commands gated CAP_SYS_ADMIN; refuse from per-DPRC ioctl that does not own the target container.
- **DPAA2 object `kfree_sensitive`** — DPNI/DPSW/DPSEC objects (holding network MACs, classifier rules, crypto keys) freed via `kfree_sensitive` to scrub on free; defense against post-free DMA read leaking network state.
- **Endpoint mismatch refuse** — `fsl_mc_get_endpoint` walks DPRC topology to verify both ends are in caller's container; cross-container endpoint walk refused; defense against VFIO-fsl-mc isolation escape.
- **MC command allow-list per UAPI** — `/dev/dprcN` accepts only DPRC-scoped read/inspect commands by default; mutate / cross-DPRC commands gated CAP_SYS_ADMIN + audit-logged.
- **GCR1 portal-stop control** — refuse to clear `GCR1_P*_STOP` while UAPI clients hold open `/dev/dprcN`; defense against runtime portal-state confusion.

Rationale: the fsl-mc bus is the privileged control plane for every NXP Layerscape platform's network and crypto data path — its objects (DPNI, DPMAC, DPSW, DPSECI) hold MAC addresses, classifier rules, crypto keys, and DMA-stream IDs. A relaxed endpoint walk that crossed DPRC boundaries would defeat VFIO-fsl-mc isolation; an unsanitized object teardown would leak crypto material via reuse; an unrestricted `/dev/dprcN` opcode set would let userland reconfigure the MC firmware. RAP/kCFI on bus ops, refcount-trapping on `fsl_mc_device`, `kfree_sensitive` on network-bearing objects, MC-command allow-listing, and CAP_SYS_ADMIN on root operations turn DPAA2 from "SoC-vendor management plane" into a structurally constrained Linux bus.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-DPxxx driver internals (dpaa2-eth, dpaa2-switch, dpaa2-qdma, dpsec) — each gets its own Tier-3 in `drivers/net/`, `drivers/dma/`, `drivers/crypto/`.
- VFIO-fsl-mc passthrough internals (covered in `drivers/vfio/fsl-mc/` Tier-3).
- MC firmware on-chip protocol details (NXP-proprietary).
- DPAA1 (legacy framework — separate driver tree).
- Implementation code
