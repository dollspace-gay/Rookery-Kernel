# Tier-3: drivers/firmware/ — kernel firmware subsystem overview

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/firmware/Makefile
  - drivers/firmware/Kconfig
  - drivers/firmware/dmi_scan.c
  - drivers/firmware/dmi-id.c
  - drivers/firmware/dmi-sysfs.c
  - drivers/firmware/edd.c
  - drivers/firmware/iscsi_ibft.c
  - drivers/firmware/memmap.c
  - drivers/firmware/qemu_fw_cfg.c
  - drivers/firmware/sysfb.c
  - drivers/firmware/sysfb_simplefb.c
  - drivers/firmware/efi/
  - drivers/firmware/arm_ffa/
  - drivers/firmware/arm_scmi/
  - drivers/firmware/arm_sdei.c
  - drivers/firmware/psci/
  - drivers/firmware/smccc/
  - drivers/firmware/google/
  - drivers/firmware/qcom/
  - drivers/firmware/imx/
  - drivers/firmware/tegra/
  - drivers/firmware/xilinx/
  - drivers/firmware/broadcom/
  - drivers/firmware/meson/
  - drivers/firmware/cirrus/
  - drivers/firmware/microchip/
  - drivers/firmware/samsung/
  - drivers/firmware/raspberrypi.c
  - drivers/firmware/stratix10-rsu.c
  - drivers/firmware/stratix10-svc.c
  - drivers/firmware/trusted_foundations.c
  - drivers/firmware/turris-mox-rwtm.c
  - drivers/firmware/ti_sci.c
  - drivers/firmware/thead,th1520-aon.c
  - drivers/firmware/mtk-adsp-ipc.c
  - include/linux/firmware.h
  - include/linux/efi.h
  - drivers/base/firmware_loader/
-->

## Summary

`drivers/firmware/` is the umbrella for every kernel subsystem that talks to platform / system firmware — runtime services, attribute tables, secure-world calls, mailbox-driven coprocessors, and DMI / SMBIOS / EDD scrape. The dominant tenant is `efi/` (UEFI runtime + variable services + Boot Services memory map + ESRT + MOK + capsule), with `arm_ffa/`, `arm_scmi/`, `psci/`, and `smccc/` covering the ARM secure-world ecosystem and per-SoC subtrees (`qcom/`, `imx/`, `tegra/`, `xilinx/`, `broadcom/`, `meson/`, `samsung/`, `microchip/`, `cirrus/`, `google/`, `raspberrypi`, `ti_sci`, `stratix10-*`, `turris-mox-rwtm`, `trusted_foundations`, `thead,th1520-aon`, `mtk-adsp-ipc`) carrying SoC-specific firmware mailbox / SCP / TZ / RPMH protocols.

This Tier-3 fixes the contract for the `drivers/firmware/` namespace itself (sysfs at `/sys/firmware/`, the `firmware_kobj`, the request-firmware loader interaction, DMI scrape, EDD scrape, memory map publication, qemu fw_cfg, sysfb / simplefb handoff) plus the per-subtree relationship table. Per-subtree internals (EFI core, EFI variables, ARM FF-A, ARM SCMI, PSCI, SMCCC, SCP firmware) get their own Tier-3 docs that descend from this one.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kobject *firmware_kobj` | `/sys/firmware/` sysfs root | `drivers::firmware::Subsystem::root_kobj` |
| `firmware_class_init()` | `firmware_class` register (request_firmware path) | covered in `drivers/base/firmware_loader.md` |
| `request_firmware(&fw, name, dev)` / `request_firmware_nowait(...)` | user-facing fw load API | covered in `drivers/base/firmware_loader.md` |
| `dmi_scan_machine()` | parse SMBIOS / DMI tables at boot | `drivers::firmware::dmi::scan_machine` |
| `dmi_get_system_info(field)` / `dmi_check_system(list)` / `dmi_first_match(list)` | per-field DMI query | `drivers::firmware::dmi::*` |
| `dmi_walk(decode, priv)` | per-entry DMI walk | `drivers::firmware::dmi::walk` |
| `edd_init()` | parse BIOS EDD (Enhanced Disk Drive) info from setup_data | `drivers::firmware::edd::init` |
| `firmware_map_add_early(start, end, type)` / `firmware_map_add_hotplug(...)` | publish per-region E820 / EFI memmap entry to `/sys/firmware/memmap/` | `drivers::firmware::memmap::add` |
| `iscsi_ibft_find()` | locate iBFT in low memory | `drivers::firmware::iscsi_ibft::find` |
| `qemu_fw_cfg_*` | QEMU paravirt config-blob device | `drivers::firmware::qemu_fw_cfg::*` |
| `sysfb_init()` / `sysfb_apply_efi_quirks()` | hand off boot framebuffer (EFI GOP / VESA) to simplefb / efifb | `drivers::firmware::sysfb::init` |
| `efi_init()` + `efi_enabled(feature)` | EFI subsystem init + per-feature query | covered in `drivers/firmware/efi-core.md` |
| `efivars_register(&efivars, &ops)` / `efivars_unregister(...)` | per-EFI-variable backend register | covered in `drivers/firmware/efi-vars.md` |
| `arm_smccc_1_1_invoke(...)` / `__arm_smccc_smc(...)` | ARM SMCCC v1.1 invocation | `drivers/firmware/smccc.md` (future) |
| `psci_invoke(fn, arg0, arg1, arg2, &res)` | PSCI 0.2/1.0 entrypoint | `drivers/firmware/psci.md` (future) |
| `scmi_handle_get(dev)` / `scmi_protocol_register(...)` | ARM SCMI handle + protocol | `drivers/firmware/arm-scmi.md` (future) |
| `ffa_dev_ops_get(dev)` | ARM FF-A device ops | `drivers/firmware/arm-ffa.md` (future) |

## Compatibility contract

REQ-1: `firmware_kobj` exposes `/sys/firmware/` as the well-known root for every firmware-published artifact (efi, dmi, memmap, edd, iscsi_ibft, ...); userspace tooling depending on `/sys/firmware/<x>/` keeps working.

REQ-2: DMI / SMBIOS scrape via `dmi_scan_machine()` is best-effort: when the SMBIOS entry-point GUID is published by EFI (`SMBIOS3_TABLE_GUID`, `SMBIOS_TABLE_GUID`) it is consumed via EFI, otherwise the legacy low-memory window `0xf0000-0xfffff` is scanned (x86 only). DMI text fields are sanitized for printable characters and exported under `/sys/class/dmi/id/`.

REQ-3: EDD scrape (x86 only) ingests `boot_params.eddbuf` populated by the boot loader / EFI stub and exports per-disk topology under `/sys/firmware/edd/int13_devXX/`.

REQ-4: `firmware_map_add_early()` mirrors the boot-time memory map (E820 + EFI memmap) to `/sys/firmware/memmap/N/{start,end,type}` for userspace memory-topology introspection; hotplug-added regions land in the same hierarchy.

REQ-5: `qemu_fw_cfg` exposes the per-blob QEMU config interface at `/sys/firmware/qemu_fw_cfg/by_key/N/` strictly when the QEMU FW_CFG signature is observed via DT or hard-coded x86 I/O ports.

REQ-6: `sysfb_init()` provides the simplefb / efifb handoff for the framebuffer the boot firmware was using; the driver releases ownership when a real display driver binds to the underlying GPU (drm_aperture handoff).

REQ-7: EFI subtree (`efi/`) is gated by `CONFIG_EFI` and is the kernel's authoritative consumer of `/sys/firmware/efi/`; per-EFI-variable filesystem (`efivarfs`) is the userspace contract.

REQ-8: ARM secure-world subtrees (`smccc/`, `psci/`, `arm_ffa/`, `arm_scmi/`, `arm_sdei.c`) are gated on `CONFIG_ARM`/`CONFIG_ARM64` and use SMC/HVC conduits; on UEFI ARM systems the SMCCC version is published via `EFI_RT_PROPERTIES_TABLE`.

REQ-9: Per-SoC firmware mailbox subtrees register through the `firmware/` root either as platform devices (DT-bound) or as their own sysfs nodes; they do not bypass `firmware_kobj`.

REQ-10: All firmware-published data is read-only from userspace by default — every write surface is opt-in (mokvar, efivars writable subset, qemu_fw_cfg DMA writes only when explicitly enabled).

REQ-11: The DMA write surface of `qemu_fw_cfg` is gated by CAP_SYS_RAWIO and platform allowlist; the boot-time selector port is read-only.

REQ-12: `request_firmware` integrates with the firmware subsystem via `drivers/base/firmware_loader/` (cross-ref) for binary blob loading from the host filesystem or built-in firmware; this Tier-3 declares the integration point but the loader internals live in the base-driver doc.

## Acceptance Criteria

- [ ] AC-1: `ls /sys/firmware/` on a UEFI x86 boot lists at least `acpi/`, `dmi/`, `efi/`, `edd/`, `memmap/`.
- [ ] AC-2: `cat /sys/class/dmi/id/sys_vendor` returns the SMBIOS vendor string (sanitized).
- [ ] AC-3: `cat /sys/firmware/memmap/0/{start,end,type}` returns the first boot-time memory region.
- [ ] AC-4: `efi-readvar` / `efivarfs` mount succeeds on a UEFI x86 boot (cross-ref `efi-vars.md`).
- [ ] AC-5: `mount -t efivarfs` succeeds and lists every UEFI variable as a file under `/sys/firmware/efi/efivars/`.
- [ ] AC-6: On ARM64 PSCI-1.1 hardware, `cat /sys/firmware/devicetree/base/psci/method` returns "smc" or "hvc".
- [ ] AC-7: `request_firmware()` from any in-tree driver succeeds with the firmware artifact under `/lib/firmware/`.
- [ ] AC-8: qemu_fw_cfg signature detection works under QEMU and `/sys/firmware/qemu_fw_cfg/by_key/` is populated.
- [ ] AC-9: kselftest `tools/testing/selftests/firmware/` passes.

## Architecture

`Subsystem` lives in `drivers::firmware::Subsystem`:

```
struct Subsystem {
  root_kobj: Arc<Kobject>,             // /sys/firmware/
  dmi: Option<KBox<dmi::Subsystem>>,
  edd: Option<KBox<edd::Subsystem>>,
  memmap: Mutex<Vec<MemmapEntry>>,     // mirror of E820/EFI memmap
  ibft: Option<KBox<IscsiIbft>>,
  fw_cfg: Option<KBox<QemuFwCfg>>,
  sysfb: Mutex<Option<SysFb>>,
  efi: Option<Arc<efi::Subsystem>>,    // covered by efi-core.md
  arm_subsys: ArmSecureWorld,          // psci/ffa/scmi/sdei/smccc dispatcher
  soc: BTreeMap<&'static str, Arc<dyn SocFirmware>>,
}
```

Boot ordering:
1. `arch_init()` populates `boot_params` / DT.
2. `efi_init()` (cross-ref `efi-core.md`) parses the EFI system table + config tables + memmap.
3. `dmi_scan_machine()` consumes `efi.smbios` (or x86 low memory fallback).
4. `firmware_map_add_early()` mirrors E820/EFI memmap into sysfs-staging.
5. `subsys_initcall(firmware_class_init)` registers `/sys/firmware/`.
6. `subsys_initcall(efisubsys_init)` registers `/sys/firmware/efi/`.
7. `device_initcall(qemu_fw_cfg_init)` / similar register the optional surfaces.
8. `sysfb_init()` hands off the boot framebuffer to simplefb/efifb.

`/sys/firmware/` layout:
- `acpi/` (cross-ref `drivers/acpi/00-overview.md`) — ACPI tables, _OSI, debugfs.
- `dmi/{entries,tables,smbios_entry_point,DMI}` — SMBIOS raw + decoded.
- `efi/{systab,fw_platform_size,fw_vendor,runtime,config_table,efivars/,mok-variables/,esrt/}` (cross-ref `efi-core.md`, `efi-vars.md`).
- `edd/int13_devXX/{host_bus,interface,version,...}` (x86 BIOS).
- `memmap/N/{start,end,type}`.
- `iscsi_ibft/` — iSCSI Boot Firmware Table.
- `qemu_fw_cfg/by_key/N/{name,size,raw}` — QEMU FW_CFG.

Per-subtree relationship summary:

| Subtree | Purpose | Tier-3 |
|---|---|---|
| `efi/` | UEFI runtime services, variables, ESRT, MOK, capsule, boot-time memmap | `efi-core.md`, `efi-vars.md` |
| `arm_ffa/` | ARM Firmware Framework for A-profile (secure-partition messaging) | `arm-ffa.md` (future) |
| `arm_scmi/` | System Control & Management Interface (perf/clk/power/sensors via mailbox) | `arm-scmi.md` (future) |
| `arm_sdei.c` | Software Delegated Exception Interface (FF-based async NMI) | `arm-sdei.md` (future) |
| `psci/` | Power State Coordination Interface (CPU on/off/suspend) | `psci.md` (future) |
| `smccc/` | SMC Calling Convention v1.x dispatcher + SoC quirks | `smccc.md` (future) |
| `google/` | Coreboot / Chromebook firmware (VBNV, GBB, VPD, GSMI, cbmem) | `google-firmware.md` (future) |
| `qcom/` | Qualcomm RPMH / SCM / TZ / QSEECOM | `qcom-firmware.md` (future) |
| `imx/`, `tegra/`, `xilinx/`, `broadcom/`, `meson/`, `samsung/`, `microchip/`, `cirrus/`, `mediatek/`, `ti_sci`, `stratix10-*`, `raspberrypi`, `turris-mox-rwtm`, `trusted_foundations`, `thead,th1520-aon` | Per-SoC firmware protocols | per-SoC Tier-3 (future) |
| `dmi-*`, `edd`, `memmap`, `qemu_fw_cfg`, `iscsi_ibft`, `sysfb*` | Architecture-neutral firmware scrape | covered here |

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dmi_walk_bounded` | OOB | DMI walker bounded by SMBIOS entry-point structure-table-length; per-entry length validated against remaining bytes. |
| `edd_disk_bounded` | OOB | per-disk descriptor index bounded by `boot_params.eddbuf_entries`. |
| `memmap_no_overlap` | UNIQUENESS | `firmware_map_add_early` rejects entries that overlap the kernel image. |
| `sysfb_handoff_no_uaf` | UAF | drm_aperture handoff drops the sysfb framebuffer before the new GPU driver maps it. |

### Layer 2: TLA+

`models/firmware/handoff.tla` (this doc): models the simplefb -> drm aperture handoff and proves that no GPU driver maps the boot-framebuffer region concurrently with the sysfb consumer.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dmi_scan_machine` post: every published `/sys/class/dmi/id/*` field is `\0`-terminated and printable | `dmi::scan_machine` |
| `firmware_map_add_early` post: every entry monotonically ordered by `start`, no overlapping regions | `memmap::add` |
| `sysfb_init` post: at most one simplefb platform device registered | `sysfb::init` |

### Layer 4: Verus/Creusot functional

EFI memory-map snapshot -> `firmware_map_add_early` walk -> `/sys/firmware/memmap/` published -> userspace reads same byte ranges as the firmware reported. Encoded as a refinement from the EFI memmap iterator (cross-ref `efi-core.md`) into the memmap kobject hierarchy.

## Hardening

- **/sys/firmware/ root is read-only by default** — every writable attribute is per-subsystem opt-in (efivars writable subset, mokvar enrollment, qemu_fw_cfg DMA).
- **DMI scrape sanitizes per-field UTF-8** — defense against control characters / ANSI escapes injected into `/sys/class/dmi/id/`.
- **EDD parse bounded** — `boot_params.eddbuf_entries` clamped to `EDDMAXNR`; per-entry length validated.
- **memmap entries validated against kernel image** — refuse to publish a memmap region that overlaps `.text`/`.rodata` lest userspace mmap it.
- **qemu_fw_cfg DMA write requires CAP_SYS_RAWIO** — and only enabled when the platform allowlist matches.
- **iscsi_ibft pointer validated** — refuse signatures whose declared length exceeds the iBFT window or whose checksum is broken.
- **simplefb handoff drops mapping atomically** — the boot-framebuffer mapping is released before any GPU driver re-claims the aperture.
- **DMI / EDD scrape rate-limited at boot** — defense against a malformed firmware table causing an O(n^2) walk.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for DMI entries, EDD records, memmap entries, and qemu_fw_cfg blobs; every user copy is `copy_to_user_with_offset` with strict size bound.
- **PAX_KERNEXEC** — every `firmware/` initcall lives in `__init` text (freed post-boot) or `__ro_after_init` text; `firmware_kobj` and per-subsystem `kobj_type` vtables are `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `request_firmware`, `dmi_walk`, `firmware_map_add_early`, and per-subsystem ioctl entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-firmware-blob references inside the firmware-loader cache; overflow trap kills the responsible task.
- **PAX_MEMORY_SANITIZE** — zero-on-free for firmware-blob buffers and `qemu_fw_cfg` DMA staging so signed-blob bytes do not bleed into reused slabs.
- **PAX_UDEREF** — SMAP/PAN enforced on every `request_firmware_into_buf` / `firmware_request_platform` entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `firmware_loader` ops, DMI walker callback table, `qemu_fw_cfg` ops, and per-SoC firmware vtables marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate `%pK` kallsyms in firmware-blob error paths behind CAP_SYSLOG; suppress per-blob physical address disclosure in dmesg.
- **GRKERNSEC_DMESG** — restrict EFI/DMI/EDD scan banners (which leak motherboard model, BIOS vendor, disk topology) to CAP_SYSLOG.
- **firmware_loader CAP_SYS_RAWIO** — `request_firmware` user surface gated; userland firmware injection (sysfs fallback) requires CAP_SYS_RAWIO + lockdown-LOCKDOWN_KERNEL_IMAGE check.
- **sysfs gate** — every writable attribute under `/sys/firmware/` has its write op behind a per-subsystem `LOCKDOWN_` check; refuse writes when lockdown is `confidentiality`.
- **BPF-LSM hook** — `security_kernel_load_data(LOADING_FIRMWARE, ...)` invoked for every firmware load; policy may veto by hash/path/source.
- **firmware-source policy** — built-in (`CONFIG_EXTRA_FIRMWARE`), filesystem (`/lib/firmware/`), and sysfs-fallback paths are independently gated; sysfs-fallback off by default.
- **SoC-firmware path quarantine** — per-SoC subtrees (qcom/imx/tegra/...) cannot reach across into `efi/` or `arm_ffa/` state; cross-tree access requires an explicit capability handle.

Rationale: `/sys/firmware/` is the canonical channel the kernel uses to expose what the platform firmware claims is true about the machine — motherboard model, secure-boot state, boot variables, mok keys, memory layout, disk topology. Any of those surfaces becoming writable without policy gating, or any of the per-SoC trees reaching across into UEFI state, would convert firmware-claims into an attacker-controlled write primitive. RAP/kCFI on the per-subsystem op tables, CAP_SYS_RAWIO on the firmware-loader sysfs-fallback, lockdown checks on every writable attribute, and BPF-LSM mediation on every firmware load turn `/sys/firmware/` from "advisory data the kernel publishes" into a structural read-mostly contract.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-subtree internals (covered in `efi-core.md`, `efi-vars.md`, and per-SoC future docs)
- `drivers/base/firmware_loader/` request_firmware internals (covered in its own Tier-3)
- ACPI subsystem (covered in `drivers/acpi/00-overview.md`)
- DT (devicetree) handling (covered in `drivers/of/` future Tier-3)
- Per-SoC mailbox / SCP firmware internals
- Implementation code
