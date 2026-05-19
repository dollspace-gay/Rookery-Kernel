# Tier-3: drivers/firmware/efi/{efi,arm-init,...}.c — EFI core (runtime services, ESRT, memory map, config tables, mokvar)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/firmware/00-overview.md
upstream-paths:
  - drivers/firmware/efi/efi.c
  - drivers/firmware/efi/efi-init.c
  - drivers/firmware/efi/arm-runtime.c
  - drivers/firmware/efi/riscv-runtime.c
  - drivers/firmware/efi/runtime-wrappers.c
  - drivers/firmware/efi/memmap.c
  - drivers/firmware/efi/memattr.c
  - drivers/firmware/efi/esrt.c
  - drivers/firmware/efi/mokvar-table.c
  - drivers/firmware/efi/reboot.c
  - drivers/firmware/efi/capsule.c
  - drivers/firmware/efi/capsule-loader.c
  - drivers/firmware/efi/fdtparams.c
  - drivers/firmware/efi/sysfb_efi.c
  - drivers/firmware/efi/efi-pstore.c
  - drivers/firmware/efi/tpm.c
  - drivers/firmware/efi/embedded-firmware.c
  - drivers/firmware/efi/cper.c
  - drivers/firmware/efi/unaccepted_memory.c
  - include/linux/efi.h
-->

## Summary

The EFI core — `efi.c` parses the UEFI system table + configuration tables + memmap, `runtime-wrappers.c` serializes every runtime-service call onto an ordered workqueue with virt-mapped-state preservation, `arm-runtime.c` / `riscv-runtime.c` provide the per-arch SetVirtualAddressMap dance, `memmap.c` / `memattr.c` own the EFI memory map + EFI Memory Attributes Table (the W^X-on-firmware-runtime contract), `esrt.c` ingests the EFI System Resource Table (per-firmware-component capsule update endpoints), `mokvar-table.c` ingests the Machine Owner Key variable table that shim leaves in memory before ExitBootServices, `reboot.c` wires `ResetSystem`, `capsule.c` / `capsule-loader.c` drive the UEFI capsule-update interface, `fdtparams.c` pulls EFI parameters from the DT for non-x86 architectures, and `unaccepted_memory.c` handles confidential-computing unaccepted-memory acceptance.

This Tier-3 covers the UEFI runtime services subsystem in totality. Per-variable storage (efivars + efivarfs) is split into `efi-vars.md`. The EFI stub (libstub) — the pre-ExitBootServices PE/COFF entry point — is out of scope here; it gets its own Tier-3.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct efi efi` | global EFI state (system-table + config tables + flags) | `drivers::efi::Subsystem` |
| `efi_init()` (arch-specific) | parse system table, config tables, memmap | `Subsystem::init` |
| `efisubsys_init()` | register `/sys/firmware/efi/`, the rts workqueue, efivars platdev | `Subsystem::register_sysfs` |
| `efi_config_parse_tables(tables, count, arch_tables)` | walk configuration-table GUIDs | `Subsystem::parse_config_tables` |
| `efi_memmap_init_early(data)` / `efi_memmap_init_late(...)` / `efi_memmap_install(...)` | per-stage memmap install (early_memremap -> memremap -> ioremap) | `Memmap::init_*` / `install` |
| `for_each_efi_memory_desc(md)` | iterate per-entry EFI memmap | `Memmap::iter` |
| `efi_mem_desc_lookup(phys, &out)` / `efi_mem_attributes(phys)` | per-pa lookup of EFI memmap entry + attributes | `Memmap::lookup` |
| `efi_memattr_init()` / `efi_memattr_apply_permissions(mm, fn)` | parse EFI Memory Attributes Table + apply W^X | `MemAttr::init` / `apply` |
| `efi_set_virtual_address_map(...)` | one-shot SetVirtualAddressMap call | `Runtime::set_virtual_address_map` |
| `__efi_call_virt(...)` / `arch_efi_call_virt_setup` / `arch_efi_call_virt_teardown` | per-call CR3/PCID switch + barrier | `Runtime::call_virt` |
| `efi_rts_work_fn(work)` + `__efi_queue_work(...)` | ordered rts workqueue dispatch | `Runtime::queue_call` |
| `efi.get_time()` / `efi.set_time(...)` / `efi.reset_system(...)` / etc. | per-runtime-service function pointers | `Runtime::*_call` |
| `efi_reboot(reboot_mode, ...)` | reboot via ResetSystem | `Reboot::reboot` |
| `efi_esrt_init()` / `efi_get_runtime_map_size()` | ESRT ingest | `Esrt::init` |
| `efi_mokvar_table_init()` / `efi_mokvar_entry_find()` / `efi_mokvar_entry_next()` | MOK variable table ingest | `MokVar::init` / `find` / `next` |
| `efi_capsule_supported(guid, flags, size, &reset)` / `efi_capsule_update(...)` | per-capsule check + submit | `Capsule::supported` / `update` |
| `efi_arch_mem_reserve(addr, size)` | per-arch boot-services-data reservation | `Subsystem::arch_mem_reserve` |
| `efi_reserve_boot_services()` / `efi_free_boot_services()` | per-arch boot-services memmap reserve + free | `Subsystem::*_boot_services` |
| `efi_query_variable_store(attr, size, nonblocking)` | per-arch variable-storage quota | `Subsystem::query_variable_store` |
| `efi_tpm_eventlog_init()` | hand TPM2 event log into IMA | `Tpm::eventlog_init` |
| `efi_accept_memory(start, end)` (TDX/SEV-SNP) | per-arch accept unaccepted memory | `Unaccepted::accept` |

## Compatibility contract

REQ-1: Per-arch `efi_init()` populates `struct efi efi` from the UEFI system table provided by the boot stub (x86: `boot_params.efi_info`; ARM/RISC-V: DT `/chosen/linux,uefi-system-table`); fields default to `EFI_INVALID_TABLE_ADDR` until found.

REQ-2: Configuration tables walked by GUID against `common_tables[]` + per-arch overrides: ACPI 2.0, ACPI, SMBIOS3, SMBIOS, ESRT, MEMATTR, RNG, TPMEventLog, TPMFinalLog, CCFinalLog, MEMRESERVE, INITRD, RTPROP, MOKvar, CocoSecret, Unaccepted, OvmfDebugLog, RCI2.

REQ-3: EFI memory map staged in three phases: `early` (early_memremap during boot), `late` (memremap into permanent kernel mapping), `install` (replace the early staging with the late one). Concurrent walks are RCU-safe across the swap.

REQ-4: Per-arch SetVirtualAddressMap is a one-shot, irrevocable call performed exactly once after the kernel virt-mapping is ready; failure permanently disables runtime services (`EFI_RUNTIME_SERVICES` flag cleared).

REQ-5: EFI Memory Attributes Table (`memattr.c`) is parsed when `EFI_MEMORY_ATTRIBUTES_TABLE_GUID` is present; per-region RO/XP attributes applied to `efi_mm` so runtime-services regions are W^X.

REQ-6: All runtime-service calls funnelled through `efi_rts_wq` (single-threaded ordered workqueue) on architectures where concurrent SMM/runtime entry is forbidden by spec; non-blocking variants (`set_variable_nonblocking`) bypass the wq when the cmdline is in atomic context.

REQ-7: ESRT exposed at `/sys/firmware/efi/esrt/entries/entry%d/` with per-entry `{fw_class,fw_type,fw_version,lowest_supported_fw_version,capsule_flags,last_attempt_version,last_attempt_status}`; the table is one-shot at boot.

REQ-8: MOK variable table parsed when `LINUX_EFI_MOK_VARIABLE_TABLE_GUID` is published by shim; per-MOK variable enumerated under `/sys/firmware/efi/mok-variables/`. Table is sealed RO after parse (`__ro_after_init`).

REQ-9: Capsule update interface accepts user-provided UEFI capsule images via `/dev/efi_capsule_loader`; per-capsule GUID checked via `QueryCapsuleCapabilities` before `UpdateCapsule`.

REQ-10: Reboot via `efi.reset_system(EFI_RESET_COLD/WARM/SHUTDOWN, EFI_SUCCESS, 0, NULL)` when `EFI_RT_SUPPORTED_RESET_SYSTEM` is set; falls back to the prior `reboot_mode` driver otherwise.

REQ-11: `EFI_RT_PROPERTIES_TABLE_GUID` consumed to gate which runtime services the firmware claims to support post-ExitBootServices; per-bit mask stored in `efi.runtime_supported_mask`.

REQ-12: Unaccepted-memory table (TDX / SEV-SNP) reserved + accepted before being handed to the buddy allocator; per-range bitmap protects against double-accept.

## Acceptance Criteria

- [ ] AC-1: `dmesg | grep -i efi` on UEFI x86 shows `EFI v2.x by <vendor>` plus the parsed `ACPI 2.0=`, `SMBIOS=`, `MOKvar=`, `MEMATTR=` lines.
- [ ] AC-2: `ls /sys/firmware/efi/` lists `systab`, `fw_platform_size`, `fw_vendor`, `runtime`, `config_table`, `efivars/`, `esrt/`.
- [ ] AC-3: `cat /sys/firmware/efi/runtime` shows the supported runtime-services mask (post-RT_PROPERTIES parse).
- [ ] AC-4: `efibootmgr -v` works (round-trips Boot#### / BootOrder variables via runtime services).
- [ ] AC-5: ESRT entries reported under `/sys/firmware/efi/esrt/entries/` correspond to firmware components reported by the OEM `fwupd` agent.
- [ ] AC-6: MOK enrollment test: `mokutil --list-enrolled` enumerates entries from `/sys/firmware/efi/mok-variables/`.
- [ ] AC-7: SetVirtualAddressMap test: after kernel virt-mapping is live, `efi.get_time()` returns sensible RTC; failure path correctly disables runtime services without crashing.
- [ ] AC-8: ARM64 PSCI + EFI: kernel boots from grub-efi, runtime services functional, `EFI_RT_PROPERTIES_TABLE` parsed.
- [ ] AC-9: Capsule update test on a capsule-capable platform: `efi_capsule_loader` write + reboot triggers firmware update.
- [ ] AC-10: TDX guest test: unaccepted memory accepted before allocation, no double-accept observed.
- [ ] AC-11: kselftest `tools/testing/selftests/firmware/efi/` (where present) passes.

## Architecture

`Subsystem` lives in `drivers::efi::Subsystem`:

```
struct Subsystem {
  efi: KCell<EfiGlobal>,             // struct efi mirror
  flags: AtomicU64,                  // EFI_BOOT / RUNTIME_SERVICES / MEMMAP / 64BIT / ...
  runtime_supported_mask: AtomicU32,
  rts_wq: Arc<OrderedWorkqueue>,
  rts_lock: Mutex<()>,               // serialize call_virt setup/teardown
  memmap: RcuCell<Memmap>,
  memattr: Once<MemAttr>,
  esrt: Once<Esrt>,
  mokvar: Once<MokVar>,
  unaccepted: Mutex<Option<Unaccepted>>,
  efi_mm: Pin<KBox<MmStruct>>,       // runtime-services virt address space
  kobj: Arc<Kobject>,                // /sys/firmware/efi/
}

struct Memmap {
  phys_map: u64,
  map: Option<NonNull<u8>>,          // memremap'd
  desc_size: u32,
  desc_version: u32,
  nr_map: usize,
  flags: u32,
}

struct MemAttr {
  table: NonNull<MemAttrTable>,      // RO
  num_entries: u32,
  desc_size: u32,
}
```

Boot ordering on x86:
1. Boot stub passes `boot_params.efi_info` (system-table phys addr, memmap phys addr, memmap size, desc size, desc version).
2. `efi_init()` early-memremaps system table, copies fields into `efi`, parses configuration tables via `efi_config_parse_tables`.
3. Each configuration table whose GUID matches `common_tables[]` populates its slot in `efi` (e.g. `efi.acpi20 = table_phys`).
4. `efi_memmap_init_early(...)` early-memremaps the memmap; `for_each_efi_memory_desc` becomes usable.
5. `reserve_unaccepted(...)` reserves the unaccepted-memory table out of the buddy allocator before any allocation runs.
6. `efi_memattr_init()` parses the MEMATTR table (if present).
7. `efi_esrt_init()` parses the ESRT.
8. `efi_mokvar_table_init()` parses the MOK variable table.
9. `efi_tpm_eventlog_init()` hands the TPM2 event log into IMA.
10. After kernel virt mapping is live, `efi_enter_virtual_mode()` runs SetVirtualAddressMap.
11. `subsys_initcall(efisubsys_init)` registers `/sys/firmware/efi/`.

SetVirtualAddressMap dance:
1. Acquire `rts_lock`.
2. Allocate `efi_mm` with userland-style page tables.
3. For each runtime-services memmap entry: map at the new virtual address (pgd → ... → pte) with attributes from MEMATTR (RO + XP per region).
4. Switch CR3 → `efi_mm.pgd`.
5. Call `efi.set_virtual_address_map(memmap_size, desc_size, desc_version, &virt_memmap)`.
6. On success: rewrite `efi.runtime->{get_time,set_time,...}` to the virtual-mapped function pointers.
7. On failure: clear `EFI_RUNTIME_SERVICES` flag, mark every `efi.<rt_service>` as `efi_unimplemented`.
8. Switch CR3 back to kernel pgd.
9. Release `rts_lock`.
10. (One-shot — never repeated.)

Per-runtime-service call (`get_time` example):
1. `efi.get_time(tm, tc)` invoked from `rtc-efi` or `efivars`.
2. Enqueue work on `efi_rts_wq` with arg pack.
3. Worker thread: switch CR3 → `efi_mm.pgd`, call the per-arch `__efi_call_virt(get_time, ...)` thunk (which saves the kernel FPU state, sets up the firmware stack, fences, then `call *fp`).
4. On return: switch CR3 back, restore FPU state, fence, propagate status.
5. Wait for worker completion (caller is a normal kernel thread).

Memory-map walker:
- `for_each_efi_memory_desc(md)` is RCU-read; per-entry stride is `desc_size` (not `sizeof(efi_memory_desc_t)` because firmware may extend).
- `efi_mem_desc_lookup(pa, &out)` linear scan with kernel-image overlap rejection.

ESRT walker:
- One-shot at boot: per-fw_class GUID -> kobj under `/sys/firmware/efi/esrt/entries/entry%d/`.
- Read-only post-init.

MOK var walker:
- Parsed from `mokvar_table` GUID region (provided by shim).
- Each entry: `{name[256], data_size, data[data_size]}`.
- Exposed RO under `/sys/firmware/efi/mok-variables/<name>/`.

Capsule loader:
- Char device `/dev/efi_capsule_loader` accepts user-written capsule blob.
- After write completion, `efi_capsule_supported(guid, flags, size, &reset)` checks firmware capabilities.
- `efi_capsule_update(...)` submits; if `reset == EFI_RESET_*`, schedule reboot.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `memmap_walk_no_oob` | OOB | `for_each_efi_memory_desc` bounded by `nr_map * desc_size`; per-entry stride respected even if firmware extends descriptor. |
| `memattr_apply_no_oob` | OOB | per-region `apply_permissions` bounded by memmap region size; refuses overlap with kernel image. |
| `rts_workqueue_no_uaf` | UAF | per-call arg struct lifetime exceeds worker completion; `flush_work` waits before drop. |
| `mokvar_walk_no_oob` | OOB | per-entry length validated; cumulative length bounded by table size. |
| `set_virt_addr_map_once` | UNIQUENESS | SetVirtualAddressMap called at most once; subsequent attempts trap. |

### Layer 2: TLA+

`models/efi/runtime_serialization.tla` (this doc): models the rts workqueue + CR3 switch sequence and proves that no two runtime-service calls execute concurrently with overlapping firmware stack.

`models/efi/svam.tla` (this doc): models SetVirtualAddressMap success and failure paths and proves that on failure all runtime function pointers are atomically demoted to `efi_unimplemented` before any caller observes the cleared `EFI_RUNTIME_SERVICES` flag.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Memmap::install` post: late mmap covers every phys region the early mmap covered, attributes preserved | `Memmap::install` |
| `MemAttr::apply` post: every runtime-services memmap region is RO+XP or RX (never RWX) in `efi_mm` | `MemAttr::apply` |
| `Runtime::call_virt` post: CR3 restored to kernel pgd before return | `Runtime::call_virt` |
| `Unaccepted::accept` post: per-range bitmap bit set; double-accept rejected with `-EALREADY` | `Unaccepted::accept` |

### Layer 4: Verus/Creusot functional

Configuration-table parse -> per-GUID slot populated -> sysfs publish -> userspace observes consistent value. Encoded as a refinement from the EFI configuration-table byte view into the typed `Subsystem` state.

## Hardening

(Inherits row-1 features from `drivers/firmware/00-overview.md` § Hardening.)

EFI core specific reinforcement:

- **EFI memmap kernel-image overlap rejection** — any descriptor whose range overlaps `.text`/`.rodata` is refused; defense against malformed memmap exposing kernel pages to firmware DMA.
- **MEMATTR W^X enforcement** — every runtime-services region mapped into `efi_mm` is RO or XP; never both writable and executable.
- **SetVirtualAddressMap one-shot** — second call attempts trap; firmware cannot be re-entered with new mappings.
- **rts workqueue ordered single-threaded** — defense against firmware bugs that assume serial invocation.
- **Capsule GUID allowlist** — per-platform capsule GUID checked via `QueryCapsuleCapabilities` before `UpdateCapsule`; refusal returns `-EOPNOTSUPP`.
- **mokvar table sealed RO post-init** — `__ro_after_init` page attribute applied after parse.
- **Unaccepted-memory per-range bitmap** — defense against double-accept and against acceptance of out-of-range pages.
- **EFI runtime workqueue rate-limited at boot** — defense against early-init runtime-service flooding stalling boot.
- **`/dev/efi_capsule_loader` per-write size bound** — defense against giant-capsule memory exhaustion.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `efi_memmap_install`, ESRT entries, MOK variable entries, capsule staging buffers; user copy strictly `copy_*_user_with_offset` with bounded size.
- **PAX_KERNEXEC** — `efi.c` core, `runtime-wrappers.c`, `memmap.c`, `memattr.c` live in `__ro_after_init` text; per-arch `__efi_call_virt` thunks placed in `.text.efi` segment marked RX.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `efi_init`, `efisubsys_init`, every runtime-service workqueue entry, and the capsule-loader write path.
- **PAX_REFCOUNT** — saturating `refcount_t` on the rts workqueue work-struct refcount and per-capsule staging-buffer refcount.
- **PAX_MEMORY_SANITIZE** — zero-on-free for capsule staging buffers, MOK variable copies, and the early-memremap scratch used during config-table parse so secret variables (Boot####, KEK) do not bleed into reused pages.
- **PAX_UDEREF** — SMAP/PAN enforced on every `efi_capsule_loader_write` and `efivarfs` entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `efi.runtime->{get_time,set_time,reset_system,get_variable,set_variable,...}` function pointers marked `__ro_after_init` (set once during SetVirtualAddressMap), with kCFI-typed indirect dispatch through the per-arch call-virt thunk.
- **GRKERNSEC_HIDESYM** — gate kallsyms in EFI fault paths behind CAP_SYSLOG; suppress `%p` of firmware function-pointers in dmesg.
- **GRKERNSEC_DMESG** — restrict EFI table-parse banners (which leak firmware-vendor, BIOS revision, MOK ownership) and any runtime-service error trace to CAP_SYSLOG.
- **SetVirtualAddressMap PAX_KERNEXEC** — the one-shot SVAM call runs with kernel `.text` mapped RO; runtime-services regions land in `efi_mm` with strict W^X per MEMATTR.
- **Runtime CR0.WP + SMAP** — every `__efi_call_virt` entry asserts `CR0.WP == 1` and `EFLAGS.AC == 0`; refuse to enter firmware with relaxed protections.
- **mokvar table RO** — table region marked `__ro_after_init`; defense against runtime tampering of MOK ownership.
- **Secure-boot lockdown** — when `LOCKDOWN_INTEGRITY_MAX` is in force, `efi.set_variable` is denied for variables in the secure-boot policy namespace (PK, KEK, db, dbx); capsule update gated by lockdown integrity-level check.
- **Unaccepted-memory bitmap RO** — per-range acceptance state stored under a refcounted lock; double-accept attempts trap rather than silently succeed.
- **ESRT entries RO** — `/sys/firmware/efi/esrt/` exposes per-entry read-only kobjs; no writable surface (firmware update goes via capsule loader gated separately).

Rationale: the EFI core is the kernel's contact surface with the platform's firmware-runtime: any compromise here demotes secure boot to advisory, lets an attacker forge boot variables, or hands firmware execution to attacker-controlled regions. SVAM running with KERNEXEC on, runtime entry asserting CR0.WP and SMAP, lockdown gating SetVariable on secure-boot namespace, MEMATTR enforced W^X on the `efi_mm`, and a sealed MOK table turn UEFI runtime from "the firmware is trusted forever" into "the firmware is trusted exactly through the contract MEMATTR and RT_PROPERTIES describe and nothing more."

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- EFI variable filesystem internals (covered in `efi-vars.md`)
- EFI stub / libstub (covered in `efi-stub.md` future Tier-3)
- EFI pstore backend (covered in `pstore.md` future Tier-3)
- TPM event-log -> IMA hand-off (covered in `tpm.md` future Tier-3)
- Per-arch boot-time UEFI handover (covered in arch-specific docs)
- Implementation code
