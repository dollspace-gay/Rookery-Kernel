# Tier-3: arch/x86/kernel/setup.c — x86 early-boot setup_arch

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: arch/x86/00-overview.md
upstream-paths:
  - arch/x86/kernel/setup.c (~1315 lines)
  - arch/x86/include/uapi/asm/bootparam.h
  - arch/x86/include/uapi/asm/setup_data.h
  - arch/x86/include/uapi/asm/e820.h
  - arch/x86/kernel/e820.c (consumed)
  - arch/x86/kernel/mpparse.c (consumed)
  - arch/x86/kernel/resource.c (consumed)
  - drivers/acpi/boot.c (consumed)
  - include/linux/memblock.h
-->

## Summary

`setup_arch(char **cmdline_p)` is the *single* x86 architecture entry point invoked from cross-arch `start_kernel()`. It runs **before** any of `mm_init`, `sched_init`, `rcu_init`, `time_init`, `cred_init`, `fork_init` — i.e. the entire generic init schedule. Its job: translate the `struct boot_params` zero-page handed up by the boot loader (or by EFI hand-off) into all the global state cross-arch code expects. Per-boot_params parsing populates `boot_cpu_data.x86_phys_bits`, `command_line[]`, `saved_video_mode`, `bootloader_type`, the EFI flag set, the framebuffer descriptor, and `ROOT_DEV`. Per-`e820__memory_setup()` ingests the BIOS E820 / EFI memmap into the kernel-owned `e820_table` and feeds it into `memblock` (the bootstrap allocator). Per-resource-tree builds the `iomem_resource` / `ioport_resource` interval trees that `/proc/iomem` and `/proc/ioports` later serve. Per-`x86_init.mpparse.find_mptable` discovers the legacy Intel MultiProcessor Specification table; per-`acpi_boot_table_init` + `acpi_boot_init` discovers ACPI MADT (Multiple APIC Description Table) for SMP topology. Per-`parse_early_param` walks `command_line[]` running every `early_param("name", fn)` registration. Per-`init_mem_mapping()` builds the direct-mapping page tables from `0..max_pfn`. Per-`max_pfn` is computed by `e820__end_of_ram_pfn()` after `mtrr_trim_uncached_memory`. Per-`dmi_setup()` runs DMI/SMBIOS string scanning for hardware quirks. Per-`x86_init.mpparse.parse_smp_cfg` triggers SMP topology enumeration. Per-`init_apic_mappings` + `io_apic_init_mappings` ioremap the LAPIC and IOAPIC MMIO ranges. Critical for: every byte of cross-arch state that depends on physical-machine probing.

This Tier-3 covers `arch/x86/kernel/setup.c` (~1315 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `setup_arch()` | per-arch boot entry | `ArchSetup::setup_arch` |
| `struct boot_params` (singleton `boot_params`) | per-zero-page state | `ArchSetup::BOOT_PARAMS` |
| `parse_boot_params()` | per-zero-page decode | `ArchSetup::parse_boot_params` |
| `parse_setup_data()` | per-setup_data linked-list walk | `ArchSetup::parse_setup_data` |
| `memblock_x86_reserve_range_setup_data()` | per-setup_data memblock reserve | `ArchSetup::reserve_setup_data` |
| `early_reserve_memory()` | per-pre-memblock reservations | `ArchSetup::early_reserve_memory` |
| `e820__memory_setup()` | per-E820 → e820_table | `E820::memory_setup` |
| `e820__memblock_setup()` | per-e820_table → memblock | `E820::memblock_setup` |
| `e820__end_of_ram_pfn()` | per-max_pfn calc | `E820::end_of_ram_pfn` |
| `e820__end_of_low_ram_pfn()` | per-max_low_pfn calc | `E820::end_of_low_ram_pfn` |
| `e820__finish_early_params()` | per-`memmap=` cmdline apply | `E820::finish_early_params` |
| `e820__reserve_resources()` | per-iomem resource emit | `E820::reserve_resources` |
| `e820__register_nosave_regions()` | per-suspend nosave | `E820::register_nosave_regions` |
| `e820__setup_pci_gap()` | per-PCI hole detect | `E820::setup_pci_gap` |
| `e820__memblock_alloc_reserved_mpc_new()` | per-MP-config reserve | `E820::reserve_mpc_new` |
| `e820__range_update()` / `_remove()` / `_add()` | per-table mutation | `E820::{range_update, range_remove, range_add}` |
| `trim_bios_range()` | per-page0 + 640K-1M trim | `ArchSetup::trim_bios_range` |
| `trim_snb_memory()` | per-SandyBridge gfx quirk | `ArchSetup::trim_snb_memory` |
| `e820_add_kernel_range()` | per-kernel-text-in-E820 audit | `ArchSetup::e820_add_kernel_range` |
| `setup_kernel_resources()` | per-text/rodata/data/bss → iomem | `ArchSetup::setup_kernel_resources` |
| `reserve_standard_io_resources()` | per-DMA/PIC/timer/keyboard ioport | `ArchSetup::reserve_standard_io_resources` |
| `arch_reserve_crashkernel()` | per-`crashkernel=` reserve | `ArchSetup::reserve_crashkernel` |
| `extend_brk()` / `reserve_brk()` | per-early-brk allocator | `ArchSetup::{extend_brk, reserve_brk}` |
| `relocate_initrd()` / `reserve_initrd()` | per-initrd placement | `ArchSetup::{relocate_initrd, reserve_initrd}` |
| `copy_edd()` | per-BIOS EDD copy | `ArchSetup::copy_edd` |
| `x86_configure_nx()` / `x86_report_nx()` | per-_PAGE_NX detect | `ArchSetup::{configure_nx, report_nx}` |
| `init_mem_mapping()` (consumed) | per-direct-map build | `Mm::init_mem_mapping` |
| `early_alloc_pgt_buf()` (consumed) | per-pgt-buf reserve | `Mm::early_alloc_pgt_buf` |
| `kernel_randomize_memory()` (consumed) | per-KASLR base | `Mm::kernel_randomize_memory` |
| `acpi_table_upgrade()` / `acpi_boot_table_init()` (consumed) | per-ACPI early-init | `Acpi::{table_upgrade, boot_table_init}` |
| `early_acpi_boot_init()` / `acpi_boot_init()` (consumed) | per-MADT scan | `Acpi::{early_boot_init, boot_init}` |
| `x86_init.mpparse.find_mptable` (consumed) | per-MP-table locate | `MpParse::find_mptable` |
| `x86_init.mpparse.early_parse_smp_cfg` (consumed) | per-MP/MADT early parse | `MpParse::early_parse_smp_cfg` |
| `x86_init.mpparse.parse_smp_cfg` (consumed) | per-MP/MADT full parse | `MpParse::parse_smp_cfg` |
| `init_apic_mappings()` (consumed) | per-LAPIC fixmap | `Apic::init_mappings` |
| `io_apic_init_mappings()` (consumed) | per-IOAPIC fixmap | `IoApic::init_mappings` |
| `check_x2apic()` (consumed) | per-x2APIC BIOS-state probe | `Apic::check_x2apic` |
| `dump_kernel_offset()` | per-panic KASLR dump | `ArchSetup::dump_kernel_offset` |
| `i386_reserve_resources()` | per-x86_32 VGA RAM resource | `ArchSetup::i386_reserve_resources` |
| `arch_cpu_is_hotpluggable(cpu)` | per-CPU-hotplug eligibility | `ArchSetup::cpu_is_hotpluggable` |
| `boot_cpu_data` (singleton) | per-BSP CPUID state | shared |
| `mmu_cr4_features` (singleton) | per-CR4 snapshot | shared |
| `max_low_pfn_mapped`, `max_pfn_mapped` | per-direct-map bounds | shared |

## Compatibility contract

REQ-1: setup_arch parameter:
- `cmdline_p`: out-pointer; on return points to `command_line[COMMAND_LINE_SIZE]` (4096) which contains a NUL-terminated copy of `boot_command_line` (possibly prepended/overridden by `CONFIG_CMDLINE`).

REQ-2: parse_boot_params (from `struct boot_params boot_params`):
- ROOT_DEV ← `old_decode_dev(hdr.root_dev)`.
- `sysfb_primary_display.screen` ← `screen_info`.
- `sysfb_primary_display.edid` ← `edid_info` (if `CONFIG_FIRMWARE_EDID`).
- `apm_info.bios` ← `apm_bios_info` (x86_32).
- `ist_info` ← `ist_info` (x86_32).
- `saved_video_mode` ← `hdr.vid_mode`.
- `bootloader_type` ← `hdr.type_of_loader`; if upper-nibble == 0xe, fold in `hdr.ext_loader_type+0x10`.
- `bootloader_version` ← (bootloader_type & 0xf) | (hdr.ext_loader_ver << 4).
- `rd_image_start` ← `hdr.ram_size & 0x07FF` (if `CONFIG_BLK_DEV_RAM`).
- EFI flags: detect `EL32` / `EL64` signature in `efi_info.efi_loader_signature`, set `EFI_BOOT` and possibly `EFI_64BIT`.
- `root_mountflags &= ~MS_RDONLY` if `!hdr.root_flags`.

REQ-3: parse_setup_data:
- Walk the linked list rooted at `boot_params.hdr.setup_data` (physical-addr next-pointer chain).
- For each entry: `early_memremap` header, dispatch on `type`:
  - `SETUP_E820_EXT`: `e820__memory_setup_extended(pa, len)`.
  - `SETUP_DTB`: `add_dtb(pa)`.
  - `SETUP_EFI`: `parse_efi_setup(pa, len)`.
  - `SETUP_IMA`: `add_early_ima_buffer(pa)` (memblock-reserves `ima_kexec_buffer_phys`).
  - `SETUP_KEXEC_KHO`: `add_kho(pa, len)` (populates KHO FDT + scratch from `struct kho_data`).
  - `SETUP_RNG_SEED`: `add_bootloader_randomness(data, len)`, then `memzero_explicit` seed + length for forward secrecy.
  - default: skip.
- `early_memunmap` after each.

REQ-4: memblock_x86_reserve_range_setup_data:
- Walk same chain; `memblock_reserve_kern(pa, sizeof(setup_data) + data->len)` for each.
- For `SETUP_INDIRECT`: also remap full `len`, follow `struct setup_indirect` and reserve `indirect->addr/len`.

REQ-5: e820 memory setup:
- `e820__memory_setup()` builds `e820_table` from `boot_params.e820_table[0..e820_entries]` (calls `x86_init.resources.memory_setup` ABI; default = `e820__memory_setup_default()`).
- Per-entry types: `E820_TYPE_RAM`, `E820_TYPE_RESERVED`, `E820_TYPE_ACPI`, `E820_TYPE_NVS`, `E820_TYPE_UNUSABLE`, `E820_TYPE_PMEM`, `E820_TYPE_PRAM`, `E820_TYPE_RESERVED_KERN`, `E820_TYPE_SOFT_RESERVED`.
- `e820__finish_early_params()` applies `memmap=` and `mem=` cmdline mutations to `e820_table`.
- `e820__memblock_setup()` walks `e820_table` and emits `memblock_add()` for `E820_TYPE_RAM` and `memblock_reserve()` for kernel-reserved ranges.

REQ-6: max_pfn / max_low_pfn / max_pfn_mapped:
- `max_pfn = e820__end_of_ram_pfn()` (page-frame number of highest RAM byte rounded up).
- Re-runs after `mtrr_trim_uncached_memory(max_pfn)` if MTRRs truncate.
- `max_possible_pfn = max_pfn`.
- x86_64: `max_low_pfn = max_pfn > (1 << (32 - PAGE_SHIFT)) ? e820__end_of_low_ram_pfn() : max_pfn`.
- x86_32: `find_low_pfn_range()` sets `max_low_pfn`.
- `max_low_pfn_mapped` / `max_pfn_mapped` (globals) populated by `init_mem_mapping()` callee.

REQ-7: early_reserve_memory order (must precede `e820__memory_setup`):
1. `memblock_reserve_kern(__pa_symbol(_text), __end_of_kernel_reserve - _text)` (kernel image).
2. `memblock_reserve(0, SZ_64K)` (BIOS-corruption guard + L1TF page-0 protection).
3. `early_reserve_initrd()` (if initrd present).
4. `memblock_x86_reserve_range_setup_data()` (REQ-4).
5. `reserve_bios_regions()` (BIOS EBDA/ROM).
6. `trim_snb_memory()` (SNB gfx bad-pages).

REQ-8: trim_bios_range:
- `e820__range_update(0, PAGE_SIZE, RAM, RESERVED)` (page 0 always reserved).
- `e820__range_remove(BIOS_BEGIN=0xa0000, BIOS_END-BIOS_BEGIN, E820_TYPE_RAM)` (640K–1M).
- `e820__update_table(e820_table)`.

REQ-9: setup_kernel_resources (post-init `iomem_resource` children):
- `code_resource`: `[__pa_symbol(_text) .. __pa_symbol(_etext)-1]`, `IORESOURCE_BUSY | IORESOURCE_SYSTEM_RAM`, name "Kernel code".
- `rodata_resource`: `[__start_rodata .. __end_rodata-1]`, "Kernel rodata".
- `data_resource`: `[_sdata .. _edata-1]`, "Kernel data".
- `bss_resource`: `[__bss_start .. __bss_stop-1]`, "Kernel bss".
- `insert_resource(&iomem_resource, ...)` for each.

REQ-10: reserve_standard_io_resources (children of `ioport_resource`):
- "dma1" [0x00..0x1f], "pic1" [0x20..0x21], "timer0" [0x40..0x43], "timer1" [0x50..0x53],
  "keyboard" [0x60..0x60], "keyboard" [0x64..0x64], "dma page reg" [0x80..0x8f],
  "pic2" [0xa0..0xa1], "dma2" [0xc0..0xdf], "fpu" [0xf0..0xff].
- All `IORESOURCE_BUSY | IORESOURCE_IO`. `request_resource(&ioport_resource, ...)`.

REQ-11: command-line handling:
- `strscpy(command_line, boot_command_line, COMMAND_LINE_SIZE=4096)`; `*cmdline_p = command_line`.
- `CONFIG_CMDLINE_BOOL` && `CONFIG_CMDLINE_OVERRIDE`: replace `boot_command_line` with `builtin_cmdline` entirely.
- `CONFIG_CMDLINE_BOOL` only: append `boot_command_line` to `builtin_cmdline` separated by space.
- `parse_early_param()` (cross-arch) walks `command_line` and invokes every `early_param("k", fn)` callback.

REQ-12: crashkernel reservation:
- `parse_crashkernel(boot_command_line, memblock_phys_mem_size(), &crash_size, &crash_base, &low_size, &cma_size, &high)`.
- `reserve_crashkernel_generic(crash_size, crash_base, low_size, high)`.
- `reserve_crashkernel_cma(cma_size)`.
- Skipped on Xen PV.
- Gated `CONFIG_CRASH_RESERVE`.

REQ-13: extend_brk / reserve_brk:
- `extend_brk(size, align)`: bumps `_brk_end` within `[__brk_base .. __brk_limit)`, panics on overflow, zeroes returned region.
- `reserve_brk()`: `memblock_reserve_kern(__pa_symbol(_brk_start), _brk_end - _brk_start)`, then sets `_brk_start = 0` to lock down further allocations.

REQ-14: NX configuration:
- `x86_configure_nx()`: `__supported_pte_mask |= _PAGE_NX` if `boot_cpu_has(X86_FEATURE_NX)` else `&= ~_PAGE_NX`.
- Must run **before** `parse_early_param()` so early EHCI-debug console's `set_fixmap()` sees correct mask.
- `x86_report_nx()`: pr_info / pr_notice based on NX support + PAE.

REQ-15: setup_arch call-graph (in order; lines 884–1278):
1. (x86_32 only) `clone_pgd_range(swapper_pg_dir, initial_page_table, KERNEL_PGD_PTRS)`; `load_cr3`; `__flush_tlb_all()`.
2. (x86_64 only) `boot_cpu_data.x86_phys_bits = MAX_PHYSMEM_BITS`.
3. CMDLINE assembly (REQ-11).
4. `olpc_ofw_detect()`.
5. `idt_setup_early_traps()`.
6. `early_cpu_init()`.
7. `jump_label_init()`.
8. `static_call_init()`.
9. `early_ioremap_init()`.
10. `setup_olpc_ofw_pgd()`.
11. `parse_boot_params()` (REQ-2).
12. `x86_init.oem.arch_setup()`.
13. `early_reserve_memory()` (REQ-7).
14. `iomem_resource.end = (1ULL << boot_cpu_data.x86_phys_bits) - 1`.
15. `e820__memory_setup()`.
16. `parse_setup_data()` (REQ-3).
17. `copy_edd()`.
18. `setup_initial_init_mm(_text, _etext, _edata, _brk_end)`.
19. `x86_configure_nx()` (REQ-14).
20. `parse_early_param()`.
21. `efi_memblock_x86_reserve_range()` (if `EFI_BOOT`).
22. `x86_report_nx()`.
23. `apic_setup_apic_calls()`.
24. `acpi_mps_check()` — if returns nonzero: `apic_is_disabled = true`; `setup_clear_cpu_cap(X86_FEATURE_APIC)`.
25. `e820__finish_early_params()`.
26. `efi_init()` (if `EFI_BOOT`).
27. `reserve_ibft_region()`.
28. `x86_init.resources.dmi_setup()` (DMI/SMBIOS scan).
29. `init_hypervisor_platform()`.
30. `tsc_early_init()`.
31. `x86_init.resources.probe_roms()`.
32. `setup_kernel_resources()` (REQ-9).
33. `e820_add_kernel_range()` (warn if kernel not in RAM in E820).
34. `trim_bios_range()` (REQ-8).
35. (x86_32 only) PPro RAM-bug 256KB hole at 1.75GB; (x86_64 only) `early_gart_iommu_check()`.
36. `max_pfn = e820__end_of_ram_pfn()` (REQ-6).
37. `cache_bp_init()`; if `mtrr_trim_uncached_memory(max_pfn)`: re-run `e820__end_of_ram_pfn()`.
38. `max_possible_pfn = max_pfn`.
39. `kernel_randomize_memory()` (KASLR).
40. `find_low_pfn_range()` (x86_32) **or** `check_x2apic()` + max_low_pfn calc (x86_64).
41. `x86_init.mpparse.find_mptable()`.
42. `early_alloc_pgt_buf()`.
43. `reserve_brk()` (REQ-13).
44. `cleanup_highmap()` (x86_64 noop on x86_32).
45. `e820__memblock_setup()`.
46. `mem_encrypt_setup_arch()`.
47. `cc_random_init()`.
48. `efi_find_mirror()`; `efi_esrt_init()`; `efi_mokvar_table_init()`; `efi_reserve_boot_services()`.
49. `e820__memblock_alloc_reserved_mpc_new()` (preallocate 4K for MP-config).
50. `setup_bios_corruption_check()` (if `CONFIG_X86_CHECK_BIOS_CORRUPTION`).
51. `x86_platform.realmode_reserve()` (AP-bringup trampoline + 1M low-mem reserve).
52. `init_mem_mapping()`.
53. `cpu_init_replace_early_idt()` (FRED or real #PF handler).
54. `mmu_cr4_features = __read_cr4() & ~X86_CR4_PCIDE`.
55. `memblock_set_current_limit(get_max_mapped())`.
56. `init_ohci1394_dma_on_all_controllers()` (if `CONFIG_PROVIDE_OHCI1394_DMA_INIT`).
57. `setup_log_buf(1)`.
58. Print "Secure boot {disabled|enabled|undetermined}" from `boot_params.secure_boot`.
59. `reserve_initrd()`.
60. `acpi_table_upgrade()`.
61. `acpi_boot_table_init()` (find + memblock-reserve RSDT/XSDT/MADT/SRAT/SLIT/...).
62. `vsmp_init()`.
63. `io_delay_init()`.
64. `early_platform_quirks()`.
65. `early_acpi_boot_init()` (parse SLIT/SRAT for NUMA, very early).
66. `x86_init.mpparse.early_parse_smp_cfg()`.
67. `x86_flattree_get_config()` (FDT config from `SETUP_DTB`).
68. `initmem_init()` (NUMA initmem + memblock-to-bootmem hand-off).
69. `dma_contiguous_reserve(max_pfn_mapped << PAGE_SHIFT)`.
70. `arch_reserve_crashkernel()` (REQ-12).
71. `early_xdbc_setup_hardware()` / `early_xdbc_register_console()` (XHCI debug port).
72. `x86_init.paging.pagetable_init()`.
73. `kasan_init()`.
74. `sync_initial_page_table()`.
75. `tboot_probe()`.
76. `map_vsyscall()`.
77. `x86_32_probe_apic()`.
78. `early_quirks()`.
79. `topology_apply_cmdline_limits_early()`.
80. `acpi_boot_init()` (full MADT parse: LAPIC entries → `topology_register_apic`; IOAPIC entries; NMI sources; interrupt-source-overrides).
81. `x86_init.mpparse.parse_smp_cfg()` (legacy MP table parse if no ACPI).
82. `init_apic_mappings()`.
83. `topology_init_possible_cpus()`.
84. `init_cpu_to_node()`.
85. `init_gi_nodes()`.
86. `io_apic_init_mappings()`.
87. `x86_init.hyper.guest_late_init()`.
88. `e820__reserve_resources()`.
89. `e820__register_nosave_regions(max_pfn)`.
90. `x86_init.resources.reserve_resources()`.
91. `e820__setup_pci_gap()`.
92. (CONFIG_VT + CONFIG_VGA_CONSOLE) `vgacon_register_screen(...)`.
93. `x86_init.oem.banner()`.
94. `x86_init.timers.wallclock_init()`.
95. `therm_lvt_init()` (before `setup_local_APIC()` which soft-disables LAPIC).
96. `mcheck_init()`.
97. `register_refined_jiffies(CLOCK_TICK_RATE)`.
98. `efi_apply_memmap_quirks()` (if `EFI_BOOT`).
99. `unwind_init()`.

REQ-16: dump_kernel_offset (panic notifier):
- If `kaslr_enabled()`: `pr_emerg("Kernel Offset: 0x%lx from 0x%lx (relocation range: 0x%lx-0x%lx)\n", kaslr_offset(), __START_KERNEL, __START_KERNEL_map, MODULES_VADDR-1)`.
- Else: `pr_emerg("Kernel Offset: disabled\n")`.
- Registered via `atomic_notifier_chain_register(&panic_notifier_list, ...)`.

REQ-17: smp_prepare_cpus initiation (out-of-tree but caused-by setup_arch):
- `setup_arch` populates `__num_possible_cpus`, `cpu_possible_mask`, `cpu_present_mask` via REQ-15 step 83 (`topology_init_possible_cpus`).
- The actual `smp_prepare_cpus()` is invoked by cross-arch `kernel_init` → `smp_init()` flow, not setup_arch itself, but every prerequisite (APIC mappings, MADT-parsed processor count, NUMA node map) is set up here.

REQ-18: arch_cpu_is_hotpluggable(cpu) — `cpu > 0` (BSP not hotpluggable on x86; gated `CONFIG_HOTPLUG_CPU`).

## Acceptance Criteria

- [ ] AC-1: `setup_arch(&cmdline)` returns with `cmdline` pointing to a NUL-terminated copy of `boot_params.hdr.cmd_line_ptr` (post CONFIG_CMDLINE munging).
- [ ] AC-2: `e820_table` after `e820__memory_setup()` matches `boot_params.e820_table[0..e820_entries]` byte-for-byte modulo BIOS-quirk fixes (`trim_bios_range`, `trim_snb_memory`).
- [ ] AC-3: `memblock` is populated such that `memblock_phys_mem_size()` equals sum of `E820_TYPE_RAM` ranges post-trim.
- [ ] AC-4: `max_pfn = pfn_up(highest E820_TYPE_RAM end)` and equals upstream output on identical `boot_params`.
- [ ] AC-5: `iomem_resource` contains "Kernel code", "Kernel rodata", "Kernel data", "Kernel bss" children with exact `__pa_symbol(_text)..` bounds.
- [ ] AC-6: `ioport_resource` contains the 10 standard-I/O children of REQ-10 with exact bounds.
- [ ] AC-7: `parse_early_param()` invokes every `early_param("k", fn)` callback exactly once in command-line key order.
- [ ] AC-8: ACPI MADT parsing (`acpi_boot_init`) populates `__num_possible_cpus` and per-LAPIC topology entries identical to upstream.
- [ ] AC-9: `init_mem_mapping()` is called **after** `max_pfn` is final and **after** `kernel_randomize_memory()`; on return `max_pfn_mapped >= max_pfn`.
- [ ] AC-10: `crashkernel=` parse honored: `reserve_crashkernel_generic` invoked iff `parse_crashkernel` returns 0 and `!xen_pv_domain()`.
- [ ] AC-11: `mmu_cr4_features` after setup_arch equals `__read_cr4() & ~X86_CR4_PCIDE` measured at step 54.
- [ ] AC-12: Panic notifier registered: panic with KASLR enabled emits "Kernel Offset: …" line.
- [ ] AC-13: Setup_data linked list traversal terminates on `pa_next == 0` and never revisits a node (idempotent on cycle = abort).
- [ ] AC-14: `SETUP_RNG_SEED` data is zeroed after `add_bootloader_randomness` (forward secrecy).
- [ ] AC-15: `reserve_brk()` runs **before** `e820__memblock_setup()`; subsequent `extend_brk()` BUGs (brk locked).

## Architecture

```
struct BootParams {                   // 4096-byte zero-page, byte-identical to upstream
  screen_info: ScreenInfo,
  apm_bios_info: ApmBiosInfo,
  tboot_addr: u64,
  ist_info: IstInfo,
  acpi_rsdp_addr: u64,
  ext_ramdisk_image: u32,
  ext_ramdisk_size: u32,
  ext_cmd_line_ptr: u32,
  edid_info: EdidInfo,
  efi_info: EfiInfo,
  alt_mem_k: u32,
  scratch: u32,
  e820_entries: u8,
  eddbuf_entries: u8,
  edd_mbr_sig_buf_entries: u8,
  kbd_status: u8,
  secure_boot: u8,                    // 0=disabled, 1=enabled, 2=undetermined
  sentinel: u8,
  edd_mbr_sig_buffer: [u32; 16],
  e820_table: [E820Entry; 128],
  eddbuf: [EddInfo; 6],
  hdr: SetupHeader,                   // bzImage header at offset 0x1F1
  ...
}

struct E820Entry { addr: u64, size: u64, type: u32 }  // E820_TYPE_*

static mut BOOT_PARAMS: BootParams;   // singleton, populated by head_64.S

struct ArchSetup;
```

`ArchSetup::setup_arch(cmdline_p: &mut *const u8)`:
- Executes the 99-step sequence in REQ-15 verbatim.
- Step 11 → `parse_boot_params` (REQ-2).
- Step 13 → `early_reserve_memory` (REQ-7).
- Step 15 → `E820::memory_setup`.
- Step 16 → `parse_setup_data` (REQ-3).
- Step 20 → `parse_early_param` (cross-arch).
- Step 32 → `setup_kernel_resources` (REQ-9).
- Step 36 → `max_pfn = E820::end_of_ram_pfn()`.
- Step 45 → `E820::memblock_setup`.
- Step 52 → `Mm::init_mem_mapping`.
- Step 61 → `Acpi::boot_table_init`.
- Step 80 → `Acpi::boot_init`.
- Step 82 → `Apic::init_mappings`.

`ArchSetup::parse_setup_data()`:
1. `pa = BOOT_PARAMS.hdr.setup_data`.
2. while `pa != 0`:
   - `data = early_memremap(pa, sizeof(SetupData))`.
   - `(len, type, next) = (data.len + sizeof(SetupData), data.type, data.next)`.
   - `early_memunmap(data, sizeof(SetupData))`.
   - match `type`:
     - `SETUP_E820_EXT` → `E820::memory_setup_extended(pa, len)`.
     - `SETUP_DTB` → `Of::add_dtb(pa)`.
     - `SETUP_EFI` → `Efi::parse_setup(pa, len)`.
     - `SETUP_IMA` → `Ima::add_early_buffer(pa)`.
     - `SETUP_KEXEC_KHO` → `Kexec::add_kho(pa, len)`.
     - `SETUP_RNG_SEED` → `Random::add_bootloader_randomness(d.data, d.len)`; zero seed + length.
     - default → skip.
   - `pa = next`.

`ArchSetup::early_reserve_memory()`:
1. `memblock_reserve_kern(__pa_symbol(_text), __end_of_kernel_reserve - _text)`.
2. `memblock_reserve(0, SZ_64K)`.
3. `early_reserve_initrd()`.
4. `memblock_x86_reserve_range_setup_data()`.
5. `reserve_bios_regions()`.
6. `trim_snb_memory()`.

`ArchSetup::setup_kernel_resources()`:
1. Populate `code_resource`, `rodata_resource`, `data_resource`, `bss_resource` with `__pa_symbol` bounds.
2. `insert_resource(&iomem_resource, ...)` × 4.

`ArchSetup::extend_brk(size, align) -> *mut u8`:
1. BUG_ON(`_brk_start == 0` || `align & (align-1)`).
2. `_brk_end = (_brk_end + align - 1) & !(align - 1)`.
3. BUG_ON(`_brk_end + size > __brk_limit`).
4. ret = `_brk_end`; `_brk_end += size`; memset(ret, 0, size).
5. return ret.

`ArchSetup::reserve_brk()`:
1. if `_brk_end > _brk_start`: `memblock_reserve_kern(__pa_symbol(_brk_start), _brk_end - _brk_start)`.
2. `_brk_start = 0` (further `extend_brk` BUGs).

`ArchSetup::dump_kernel_offset(self, v, p) -> i32` (panic-notifier):
1. if `Kaslr::enabled()`: `pr_emerg("Kernel Offset: 0x%lx from 0x%lx (relocation range: 0x%lx-0x%lx)\n", Kaslr::offset(), __START_KERNEL, __START_KERNEL_map, MODULES_VADDR-1)`.
2. else: `pr_emerg("Kernel Offset: disabled\n")`.
3. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `setup_arch_cmdline_nul_terminated` | INVARIANT | per-setup_arch: on return `command_line[strnlen(command_line, 4096)] == 0`. |
| `boot_params_singleton_read_only_after_parse` | INVARIANT | per-parse_boot_params: post-call `BOOT_PARAMS` no longer mutated by setup_arch. |
| `setup_data_walk_terminates` | INVARIANT | per-parse_setup_data: walk follows `next` pointers and terminates (no cycle). |
| `extend_brk_within_bounds` | INVARIANT | per-extend_brk: returned pointer in `[__brk_base, __brk_limit - size]`. |
| `reserve_brk_locks` | INVARIANT | per-reserve_brk: post-call `_brk_start == 0`. |
| `max_pfn_monotone` | INVARIANT | per-setup_arch: `max_pfn >= e820__end_of_ram_pfn()` always after step 36. |
| `iomem_resource_children_disjoint` | INVARIANT | per-setup_kernel_resources: code/rodata/data/bss insertions non-overlapping. |
| `nx_mask_set_before_parse_early_param` | INVARIANT | per-setup_arch: `x86_configure_nx` runs before `parse_early_param`. |
| `init_mem_mapping_after_max_pfn` | INVARIANT | per-setup_arch: `init_mem_mapping()` happens after `max_pfn` is final. |
| `rng_seed_zeroed` | INVARIANT | per-parse_setup_data SETUP_RNG_SEED: `data.data` and `data.len` zeroed post-consume. |

### Layer 2: TLA+

`arch/x86/setup.tla`:
- Steps modelled as TLA+ actions: ParseBootParams, EarlyReserveMemory, E820MemorySetup, ParseSetupData, ParseEarlyParam, MaxPfnCalc, E820MemblockSetup, InitMemMapping, AcpiBootInit, ApicInitMappings.
- Properties:
  - `safety_step_order_strict` — per-setup_arch: step *i* precedes step *j* whenever *i < j* in REQ-15.
  - `safety_max_pfn_set_before_init_mem_mapping` — REQ-15 step 36 < step 52.
  - `safety_reserve_brk_before_memblock_setup` — REQ-15 step 43 < step 45.
  - `safety_parse_early_param_after_e820_setup` — REQ-15 step 15 < step 20.
  - `safety_acpi_boot_table_init_before_boot_init` — REQ-15 step 61 < step 80.
  - `safety_apic_setup_apic_calls_before_init_mappings` — REQ-15 step 23 < step 82.
  - `liveness_setup_arch_eventually_returns` — per-setup_arch: terminates (all 99 steps complete).
  - `liveness_setup_data_walk_terminates` — per-parse_setup_data: walks bounded chain.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ArchSetup::setup_arch` post: `*cmdline_p` is valid CStr | `ArchSetup::setup_arch` |
| `ArchSetup::parse_boot_params` post: `ROOT_DEV`, `bootloader_type`, `bootloader_version` derived | `ArchSetup::parse_boot_params` |
| `ArchSetup::parse_setup_data` post: every node visited exactly once | `ArchSetup::parse_setup_data` |
| `ArchSetup::extend_brk` post: ret ∈ `[_brk_start, _brk_end)` and ret % align == 0 | `ArchSetup::extend_brk` |
| `ArchSetup::reserve_brk` post: `memblock_reserve_kern(_brk_start, _brk_end - _brk_start)` invoked iff non-empty | `ArchSetup::reserve_brk` |
| `ArchSetup::setup_kernel_resources` post: 4 children inserted under `iomem_resource` | `ArchSetup::setup_kernel_resources` |
| `ArchSetup::reserve_standard_io_resources` post: 10 children inserted under `ioport_resource` | `ArchSetup::reserve_standard_io_resources` |
| `ArchSetup::trim_bios_range` post: e820_table page-0 RESERVED, [640K, 1M) RAM removed | `ArchSetup::trim_bios_range` |
| `ArchSetup::configure_nx` post: `__supported_pte_mask` has correct `_PAGE_NX` bit | `ArchSetup::configure_nx` |
| `ArchSetup::reserve_crashkernel` post: at most one reservation | `ArchSetup::reserve_crashkernel` |

### Layer 4: Verus/Creusot functional

`Per-setup_arch: boot_params → e820_table → memblock → max_pfn → init_mem_mapping → iomem/ioport resource trees → ACPI MADT-parsed topology` semantic equivalence: per-`Documentation/arch/x86/boot.rst`, `Documentation/arch/x86/zero-page.rst`, ACPI 6.5 § 5.2.12 (MADT).

## Hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

setup_arch reinforcement:

- **Per-zero-page parsed-once, then read-only** — defense against per-boot_params TOCTOU.
- **Per-setup_data walk visited-set bounded** — defense against per-malicious cycle DoS.
- **Per-SETUP_RNG_SEED zeroed post-consume** — defense against per-seed reuse / forward-secrecy loss.
- **Per-page0 always reserved** — defense against per-L1TF page-0 secret leak.
- **Per-low-64K reserved** — defense against per-BIOS low-memory corruption.
- **Per-brk locked post-`reserve_brk`** — defense against per-late-brk overwrite of memblock.
- **Per-`memmap=` / `mem=` cmdline parse on `e820_table` clone** — defense against per-malformed cmdline e820 corruption.
- **Per-crashkernel skip-on-Xen-PV** — defense against per-Xen-pv crashkernel-clobber.
- **Per-iomem/ioport BUSY flag on standard resources** — defense against per-userspace-mmio-claim.
- **Per-init_mem_mapping after KASLR + max_pfn final** — defense against per-KASLR-aware page-table corruption.
- **Per-`acpi_mps_check` failure → APIC disabled** — defense against per-broken-firmware APIC misroute.
- **Per-`mmu_cr4_features` masked of PCIDE** — defense against per-trampoline-CR4 invalid in non-long-mode.
- **Per-`dump_kernel_offset` panic-notifier registered** — defense against per-KASLR-aware crash forensic loss.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — none in `setup_arch` (no user pointers at boot), but the same bounds-check policy applies to the post-init `/proc/iomem`, `/proc/ioports`, and `/sys/firmware/efi/*` readers seeded here.
- **PAX_KERNEXEC** — early page-table built with W^X: `.text`/`.rodata` mapped RX/RO, `.data`/`.bss` mapped NX; trampoline / real-mode pages flipped RX-only after `init_real_mode`.
- **PAX_RANDMMAP** — KASLR base randomization (`kernel_randomize_memory`) drives the same entropy budget reused by user-mmap randomization later; bootparams that try to disable it are dropped under lockdown.
- **PAX_REFCOUNT** — refcounts on `iomem_resource` / `ioport_resource` saturating; setup-time resource trees cannot wrap.
- **PAX_MEMORY_SANITIZE** — `SETUP_RNG_SEED` and any in-place command-line copies poisoned (`memzero_explicit`) post-consumption; e820 stash buffers wiped before being marked read-only.
- **PAX_UDEREF** — N/A at boot; later runtime entry paths (boot CPU's first syscall) inherit SMAP from `cr4_init`.
- **PAX_RAP / kCFI** — early `x86_platform_ops`, `x86_init.*`, and `pv_ops.*` vtables installed once, then constified by `mark_rodata_ro`.
- **GRKERNSEC_KERN_LOCKDOWN** — `lockdown=integrity` enforced from `setup_arch` onward: blocks `mem=`, `memmap=`, `iomem=relaxed`, kexec without sig, and `/dev/mem`/`/dev/kmem` (see GRKERNSEC_KMEM).
- **GRKERNSEC_HIDESYM** — KASLR offset, `_text`, `_etext`, `init_top_pgt` never logged at runtime; only `dump_kernel_offset` panic-notifier exposes it post-crash, gated by `kptr_restrict`.
- **GRKERNSEC_DMESG** — early `pr_info` traces of e820/efi/acpi tables gated by `dmesg_restrict` once user-mode is up.
- **GRKERNSEC_EFI** — EFI runtime services called only under `efi_enter_virtual_mode`; runtime pool allocations marked RO post-`mark_rodata_ro`; SetVariable filtered against deny-list.
- **GRKERNSEC_BOOT** — `boot_params` zero-page parsed once, sealed RO immediately after; later writes oops.

Per-doc rationale: `setup_arch` is the one-shot fountain from which every other subsystem inherits its memory map, command line, and policy posture; reinforcing lockdown, RANDMMAP entropy, EFI-runtime isolation, and zero-page seal here means later subsystems (vDSO, signal, block, crypto) start from a verifiably hardened baseline rather than retro-fitting checks.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `arch/x86/kernel/e820.c` implementation (covered indirectly; symbol-level contracts here)
- `arch/x86/kernel/mpparse.c` MP-table format (covered in `kernel-platform.md` Tier-3 § MP/MADT)
- `arch/x86/kernel/apic/apic.c` LAPIC programming (covered in `arch/x86/apic.md` Tier-3)
- `arch/x86/kernel/smpboot.c` AP-bringup (covered in `boot.md` Tier-3 § AP bringup)
- `arch/x86/mm/init.c` `init_mem_mapping()` body (covered in `mm-init.md` Tier-3)
- `arch/x86/mm/kaslr.c` `kernel_randomize_memory()` (covered in `mm-init.md` Tier-3)
- `drivers/acpi/boot.c` ACPI early-init body (covered in `kernel-platform.md` Tier-3 § ACPI)
- `arch/x86/boot/*` real-mode setup (covered in `boot.md` Tier-3)
- `arch/x86/kernel/head_64.S` long-mode entry (covered in `boot.md` Tier-3 § Long-mode entry)
- Implementation code
