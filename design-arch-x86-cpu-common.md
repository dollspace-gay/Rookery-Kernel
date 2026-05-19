---
title: "Tier-3: arch/x86/kernel/cpu/common.c — x86 per-CPU identification and init"
tags: ["tier-3", "arch", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`arch/x86/kernel/cpu/common.c` is the shared backbone of x86 CPU bring-up. Per-boot it runs `early_cpu_init -> early_identify_cpu(&boot_cpu_data)` to fill the boot CPU's `struct cpuinfo_x86` (vendor, family/model/stepping, capability bitmap, address sizes, NULL-seg quirks, speculative-control synthetics, command-line cap overrides) before the rest of the kernel reads CPU state. Per-AP-bringup `identify_secondary_cpu(cpu)` copies `boot_cpu_data` as a template, runs `identify_cpu(c)` to re-derive the AP's full capability set, applies SMT/spec-control AP tail (`x86_spec_ctrl_setup_ap`, `update_srbds_msr`, `update_gds_msr`, `tsx_ap_init`), and marks the AP `c->initialized = true`. Per-boot vendor dispatch is via `get_cpu_vendor(c)` matching `c->x86_vendor_id` against `cpu_devs[]` (Intel / AMD / Hygon / Cyrix / Centaur / Zhaoxin / VIA / Transmeta / NSC + default) — selecting the per-vendor `struct cpu_dev` (`c_early_init` / `c_bsp_init` / `c_identify` / `c_init`). Per-CPU control-register init lives in `cr4_init()` plus the pinning machinery (`native_write_cr0` / `native_write_cr4` / `setup_cr_pinning`) that locks down WP / SMEP / SMAP / UMIP / FSGSBASE bits at runtime. Per-CPU descriptor tables: the boot GDT template `gdt_page` (per-CPU page-aligned), `switch_gdt_and_percpu_base(cpu)` to swap from early per-CPU mapping to final, `load_direct_gdt` / `load_fixmap_gdt` and `tss_setup_ist` (IST stacks for #DF / #NMI / #DB / #MCE / #VC). Per-CPU exception path activation is `cpu_init_exception_handling(boot_cpu)`. Final per-CPU state barrier is `cpu_init()` (reload FS/GS, syscall_init, mmgrab(init_mm), TLB flush, load_TR_desc). Critical for: deterministic CPU feature surface, ABI-stable `boot_cpu_data`, GDT/TSS/IDT integrity, spec-ctrl AP coherence.

This Tier-3 covers `arch/x86/kernel/cpu/common.c` (~2670 lines).

### Acceptance Criteria

- [ ] AC-1: `early_cpu_init()` populates `CPU_DEVS` from registered vendor tables and invokes `early_identify_cpu(&boot_cpu_data)`.
- [ ] AC-2: `cpu_detect()` fills `c.cpuid_level`, `c.x86_vendor_id`, `c.x86`, `c.x86_model`, `c.x86_stepping`, `c.x86_clflush_size` from CPUID(0)/(1).
- [ ] AC-3: `get_cpu_vendor()` matches `c.x86_vendor_id` against `cpu_devs[]` and assigns `this_cpu` + `c.x86_vendor`; unknown vendor falls back to `default_cpu` with `X86_VENDOR_UNKNOWN`.
- [ ] AC-4: `get_cpu_cap()` populates all NCAPINTS leafs from CPUID; `apply_forced_caps()` applies `cpu_caps_cleared` AND-NOT and `cpu_caps_set` OR.
- [ ] AC-5: `init_speculation_control()` synthesises X86_FEATURE_IBRS / IBPB / STIBP / SSBD / MSR_SPEC_CTRL from Intel + AMD primary bits.
- [ ] AC-6: `identify_cpu()` ANDs secondary CPU capability into `boot_cpu_data` (common-set semantics) and ORs bug bits back into the AP.
- [ ] AC-7: `identify_secondary_cpu()` copies `boot_cpu_data` to per-CPU `cpu_data(cpu)` only on first bring-up (`!c.initialized`), then sets `c.cpu_index = cpu` and `c.initialized = true` only after full identify completes.
- [ ] AC-8: `native_write_cr0()` re-loops to restore X86_CR0_WP when pinned bit observed missing; `native_write_cr4()` restores `cr4_pinned_bits` under `cr4_pinned_mask` and `WARN_ONCE`s.
- [ ] AC-9: `cr4_init()` ORs in `X86_CR4_PCIDE` when PCID is supported and applies pinned bits; updates `cpu_tlbstate.cr4` shadow.
- [ ] AC-10: `switch_gdt_and_percpu_base(cpu)` loads the direct (RW) GDT and writes `MSR_GS_BASE` (x86_64) or reloads `%fs = __KERNEL_PERCPU` (x86_32).
- [ ] AC-11: `tss_setup_ist()` populates DF / NMI / DB / MCE / VC IST slots from per-CPU exception stacks (x86_64 only).
- [ ] AC-12: `cpu_init_exception_handling(boot_cpu)` programs the per-CPU TSS, loads TR, sets up GHCB, and either loads the IDT or initialises FRED RSPs.
- [ ] AC-13: `cpu_init()` flushes the TLB, grabs `init_mm`, loads `sp0` to the entry trampoline stack, and (on x86_64) zeroes `FS_BASE` / `KERNEL_GS_BASE` plus calls `syscall_init()`.
- [ ] AC-14: `identify_boot_cpu()` tail invokes `setup_cr_pinning`, `x86_virt_init`, `tsx_init`, `tdx_init`, `lkgs_init` in that order after `cpu_detect_tlb`.
- [ ] AC-15: `microcode_check()` diffs stored vs current `x86_capability` post-microcode-reload and `pr_warn`s on mismatch.

### Architecture

```
struct CpuInfoX86 {
  x86_vendor: u8,                                     // X86_VENDOR_*
  x86: u8,
  x86_model: u8,
  x86_stepping: u8,
  x86_clflush_size: u16,
  x86_cache_alignment: u32,
  x86_phys_bits: u8,
  x86_virt_bits: u8,
  cpuid_level: i32,
  extended_cpuid_level: u32,
  x86_capability: [u32; NCAPINTS + NBUGINTS],
  x86_vendor_id: [u8; 16],
  x86_model_id: [u8; 64],
  x86_cache_size: u32,
  loops_per_jiffy: u64,
  cpu_index: u32,
  initialized: bool,
  // topology: cpu_die_id, cpu_core_id, ...
}

struct CpuDev {
  c_vendor: &'static str,
  c_ident: [Option<&'static str>; 2],
  c_early_init: Option<fn(&mut CpuInfoX86)>,
  c_bsp_init: Option<fn(&mut CpuInfoX86)>,
  c_init: Option<fn(&mut CpuInfoX86)>,
  c_identify: Option<fn(&mut CpuInfoX86)>,
  c_x86_vendor: u8,
}
```

`CpuCommon::early_cpu_init()`:
1. init_cpu_devs() — walk vendor cpu_dev registrations (linker section), populate CPU_DEVS[0..X86_VENDOR_NUM].
2. early_identify_cpu(&mut boot_cpu_data).

`CpuCommon::early_identify_cpu(c)`:
1. memset c.x86_capability; c.extended_cpuid_level = 0.
2. if !cpuid_feature(): identify_cpu_without_cpuid(c).
3. if cpuid_feature():
   - cpu_detect(c).
   - get_cpu_vendor(c).
   - intel_unlock_cpuid_leafs(c).
   - get_cpu_cap(c).
   - setup_force_cpu_cap(X86_FEATURE_CPUID).
   - get_cpu_address_sizes(c).
   - cpu_parse_early_param() — clearcpuid / setcpuid.
   - cpu_init_topology(c).
   - if this_cpu.c_early_init.is_some(): invoke.
   - c.cpu_index = 0.
   - filter_cpuid_features(c, warn=false).
   - check_cpufeature_deps(c).
   - if this_cpu.c_bsp_init.is_some(): invoke.
4. setup_force_cpu_cap(X86_FEATURE_ALWAYS).
5. cpu_set_bug_bits(c).
6. sld_setup(c).
7. if x86_32: clear_cpu_cap(PCID); clear_cpu_cap(SYSCALL32).
8. if !pgtable_l5_enabled(): clear_cpu_cap(LA57).
9. detect_nopl().
10. mca_bsp_init(c).

`CpuCommon::identify_cpu(c)`:
1. Reset c (loops_per_jiffy, x86_cache_size = 0, vendor = UNKNOWN, model = stepping = 0, vendor_id[0] = '\0', model_id[0] = '\0').
2. x86_64 defaults (clflush=64, phys=36, virt=48) / x86_32 defaults.
3. x86_cache_alignment = x86_clflush_size.
4. memset x86_capability + vmx_capability.
5. generic_identify(c).
6. cpu_parse_topology(c).
7. if this_cpu.c_identify: invoke.
8. apply_forced_caps(c).  // first pass
9. set_cpu_cap(c, X86_FEATURE_APIC_MSRS_FENCE) — default; AMD/Hygon clear later.
10. if this_cpu.c_init: invoke (vendor canonicalisation).
11. bus_lock_init().
12. squash_the_stupid_serial_number(c).
13. setup_smep(c); setup_smap(c); setup_umip(c).
14. filter_cpuid_features(c, warn=true).
15. check_cpufeature_deps(c).
16. if c.x86_model_id is empty: lookup or sprintf "%02x/%02x".
17. x86_init_rdrand(c); setup_pku(c); setup_cet(c).
18. apply_forced_caps(c).  // second pass
19. if c != &boot_cpu_data:
    - AND boot_cpu_data.x86_capability[0..NCAPINTS] with c.x86_capability.
    - OR boot_cpu_data.x86_capability[NCAPINTS..NCAPINTS+NBUGINTS] into c.
20. ppin_init(c).
21. mcheck_cpu_init(c).
22. numa_add_cpu(smp_processor_id()).

`CpuCommon::identify_secondary_cpu(cpu)`:
1. c = &mut PER_CPU_INFO[cpu].
2. if !c.initialized: *c = boot_cpu_data.clone().
3. c.cpu_index = cpu.
4. identify_cpu(c).
5. if x86_32: enable_sep_cpu().
6. x86_spec_ctrl_setup_ap().
7. update_srbds_msr().
8. if boot_cpu_has_bug(X86_BUG_GDS): update_gds_msr().
9. tsx_ap_init().
10. c.initialized = true.

`CpuCommon::get_cpu_vendor(c)`:
1. For i in 0..X86_VENDOR_NUM:
   - dev = CPU_DEVS[i]?  // break on None
   - id = &c.x86_vendor_id.
   - if eq(id, dev.c_ident[0]) ∨ (dev.c_ident[1].is_some() ∧ eq(id, dev.c_ident[1])):
     - current_vendor = Some(dev).
     - c.x86_vendor = dev.c_x86_vendor.
     - return.
2. pr_err_once("CPU: vendor_id '{}' unknown, using generic init.\nCPU: Your system may be unstable.", id).
3. c.x86_vendor = X86_VENDOR_UNKNOWN.
4. current_vendor = Some(&DEFAULT_CPU).

`CpuCommon::init_speculation_control(c)`:
1. if c.has(SPEC_CTRL): set(IBRS); set(IBPB); set(MSR_SPEC_CTRL).
2. if c.has(INTEL_STIBP): set(STIBP).
3. if c.has(SPEC_CTRL_SSBD) ∨ c.has(VIRT_SSBD): set(SSBD).
4. if c.has(AMD_IBRS): set(IBRS); set(MSR_SPEC_CTRL).
5. if c.has(AMD_IBPB): set(IBPB).
6. if c.has(AMD_STIBP): set(STIBP); set(MSR_SPEC_CTRL).
7. if c.has(AMD_SSBD): set(SSBD); set(MSR_SPEC_CTRL); clear(VIRT_SSBD).

`CpuCommon::cr4_init()`:
1. cr4 = __read_cr4().
2. if boot_cpu_has(PCID): cr4 |= X86_CR4_PCIDE.
3. if cr_pinning enabled: cr4 = (cr4 & ~cr4_pinned_mask) | cr4_pinned_bits.
4. __write_cr4(cr4).
5. this_cpu_write(cpu_tlbstate.cr4, cr4).

`CpuCommon::write_cr4(val)`:
1. set_register: asm "mov val, %cr4".
2. if cr_pinning enabled ∧ (val & cr4_pinned_mask) ≠ cr4_pinned_bits:
   - bits_changed = (val & cr4_pinned_mask) ^ cr4_pinned_bits.
   - val = (val & ~cr4_pinned_mask) | cr4_pinned_bits.
   - goto set_register.
3. WARN_ONCE(bits_changed ≠ 0, "pinned CR4 bits changed: 0x{bits_changed:x}!?").

`CpuCommon::cpu_init_exception_handling(boot_cpu)`:
1. tss = this_cpu_ptr(&cpu_tss_rw).
2. cpu = raw_smp_processor_id().
3. setup_getcpu(cpu).
4. if !X86_FEATURE_FRED: tss_setup_ist(tss).
5. tss_setup_io_bitmap(tss).
6. set_tss_desc(cpu, &get_cpu_entry_area(cpu).tss.x86_tss).
7. load_TR_desc().
8. setup_ghcb().
9. if X86_FEATURE_FSGSBASE: cr4_set_bits(X86_CR4_FSGSBASE); elf_hwcap2 |= HWCAP2_FSGSBASE.
10. if X86_FEATURE_FRED:
    - if !boot_cpu: cpu_init_fred_exceptions().
    - cpu_init_fred_rsps().
   else:
    - load_current_idt().

`CpuCommon::cpu_init()`:
1. cur = current.
2. cpu = raw_smp_processor_id().
3. NUMA: if numa_node == 0 ∧ early_cpu_to_node(cpu) ≠ NUMA_NO_NODE: set_numa_node.
4. if x86_64 ∨ VME ∨ TSC ∨ DE: cr4_clear_bits(VME|PVI|TSD|DE).
5. if x86_64:
   - loadsegment(fs, 0).
   - memset cur.thread.tls_array.
   - syscall_init().
   - wrmsrq(FS_BASE, 0); wrmsrq(KERNEL_GS_BASE, 0); barrier().
   - x2apic_setup(); intel_posted_msi_init().
6. mmgrab(&init_mm); cur.active_mm = &init_mm; BUG_ON(cur.mm.is_some()).
7. initialize_tlbstate_and_flush(); enter_lazy_tlb(&init_mm, cur).
8. load_sp0(cpu_entry_stack(cpu) + 1).
9. load_mm_ldt(&init_mm).
10. initialize_debug_regs(); dbg_restore_debug_regs().
11. doublefault_init_cpu_tss().
12. if is_uv_system(): uv_cpu_init().
13. load_fixmap_gdt(cpu).

### Out of Scope

- Per-vendor `c_init` bodies for Intel / AMD / Hygon / Cyrix / Centaur / Zhaoxin (covered in `arch/x86/cpu-vendor.md` Tier-3, if expanded)
- Mitigation selection (`cpu_select_mitigations`, `cpu_bugs_smt_update`) — covered in `arch/x86/cpu-mitigations.md`
- TSX / TDX / LKGS initialisation bodies — covered in `arch/x86/cpu-tsx.md` / `cpu-tdx.md` (if expanded)
- Topology parsing (`cpu_parse_topology`, `cpu_init_topology`) — covered in `arch/x86/topology.md` (if expanded)
- Microcode loader (`amd_check_microcode`, `intel_microcode_*`) — covered in `arch/x86/microcode.md` (if expanded)
- FRED bring-up bodies — covered in `arch/x86/fred.md` (if expanded)
- MCA / MCE init bodies (`mca_bsp_init`, `mcheck_cpu_init`) — covered in `arch/x86/mca.md` (if expanded)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cpuinfo_x86` | per-CPU descriptor (vendor, family, model, capability) | `CpuInfoX86` |
| `struct cpu_dev` | per-vendor ops table | `CpuDev` |
| `boot_cpu_data` | per-boot global `cpuinfo_x86` | `CpuCommon::boot_cpu_data` |
| `cpu_info` (DEFINE_PER_CPU) | per-CPU `cpuinfo_x86` | `CpuCommon::PER_CPU_INFO` |
| `cpu_devs[X86_VENDOR_NUM]` | per-vendor dispatch | `CpuCommon::CPU_DEVS` |
| `this_cpu` | per-current vendor pointer | `CpuCommon::current_vendor` |
| `gdt_page` (DEFINE_PER_CPU_PAGE_ALIGNED) | per-CPU GDT | `CpuCommon::PER_CPU_GDT` |
| `early_cpu_init()` | per-boot vendor table init + early ident | `CpuCommon::early_cpu_init` |
| `init_cpu_devs()` | per-boot populate `cpu_devs[]` | `CpuCommon::init_cpu_devs` |
| `early_identify_cpu()` | per-boot fill of boot_cpu_data | `CpuCommon::early_identify_cpu` |
| `identify_cpu()` | per-CPU full identification | `CpuCommon::identify_cpu` |
| `identify_boot_cpu()` | per-boot tail (cr_pinning, tsx, tdx) | `CpuCommon::identify_boot_cpu` |
| `identify_secondary_cpu()` | per-AP ident + spec-ctrl tail | `CpuCommon::identify_secondary_cpu` |
| `generic_identify()` | per-CPU vendor-agnostic ident | `CpuCommon::generic_identify` |
| `cpu_detect()` | per-CPU CPUID(0)/(1) family/model | `CpuCommon::cpu_detect` |
| `get_cpu_vendor()` | per-CPU vendor-string match | `CpuCommon::get_cpu_vendor` |
| `get_cpu_cap()` | per-CPU CPUID capability harvest | `CpuCommon::get_cpu_cap` |
| `get_cpu_address_sizes()` | per-CPU phys/virt bits | `CpuCommon::get_cpu_address_sizes` |
| `filter_cpuid_features()` | per-CPU prune unsupported | `CpuCommon::filter_cpuid_features` |
| `apply_forced_caps()` | per-CPU cmdline overrides | `CpuCommon::apply_forced_caps` |
| `init_speculation_control()` | per-CPU IBRS/STIBP/SSBD synthetic flags | `CpuCommon::init_speculation_control` |
| `cpu_set_bug_bits()` | per-CPU vulnerability flagging | `CpuCommon::cpu_set_bug_bits` |
| `native_write_cr0()` / `native_write_cr4()` | per-CR pinning guard | `CpuCommon::write_cr0` / `write_cr4` |
| `cr4_init()` | per-CPU CR4 PCID + pinned bits | `CpuCommon::cr4_init` |
| `cr4_update_irqsoff()` | per-CPU masked CR4 update | `CpuCommon::cr4_update_irqsoff` |
| `setup_cr_pinning()` | per-boot CR pinning enable | `CpuCommon::setup_cr_pinning` |
| `load_direct_gdt()` / `load_fixmap_gdt()` | per-CPU GDT load | `CpuCommon::load_direct_gdt` / `_fixmap` |
| `switch_gdt_and_percpu_base()` | per-AP percpu base swap | `CpuCommon::switch_gdt_and_percpu_base` |
| `tss_setup_ist()` | per-CPU IST stacks | `CpuCommon::tss_setup_ist` |
| `tss_setup_io_bitmap()` | per-CPU IO permission map | `CpuCommon::tss_setup_io_bitmap` |
| `setup_getcpu()` | per-CPU `cpunode` GDT entry | `CpuCommon::setup_getcpu` |
| `cpu_init_exception_handling()` | per-CPU TSS + IDT load | `CpuCommon::cpu_init_exception_handling` |
| `cpu_init()` | per-CPU state barrier | `CpuCommon::cpu_init` |
| `syscall_init()` | per-CPU SYSCALL/SYSENTER MSRs | `CpuCommon::syscall_init` |
| `setup_smep` / `setup_smap` / `setup_umip` / `setup_pku` / `setup_cet` | per-CPU CR4 mit | `CpuCommon::setup_*` |
| `print_cpu_info()` | per-CPU klog line | `CpuCommon::print_cpu_info` |
| `cpu_caps_cleared[]` / `cpu_caps_set[]` | cmdline cap overrides | `CpuCommon::CAPS_CLEARED` / `_SET` |
| `store_cpu_caps()` / `microcode_check()` | per-microcode-reload diff | `CpuCommon::store_cpu_caps` / `microcode_check` |

### compatibility contract

REQ-1: struct cpuinfo_x86 layout:
- x86_vendor: u8 (X86_VENDOR_INTEL=0, AMD=2, HYGON=9, CENTAUR=5, CYRIX=1, NSC=4, TRANSMETA=7, ZHAOXIN=8, VORTEX=12, UMC=3, ...).
- x86: u8 (CPUID family).
- x86_model: u8.
- x86_stepping: u8.
- x86_clflush_size: u16 (default 64 on x86_64, 32 on x86_32).
- x86_cache_alignment: u32.
- x86_phys_bits: u8 (default 36 on x86_64, 32 on x86_32; updated by `get_cpu_address_sizes`).
- x86_virt_bits: u8 (default 48 / 32).
- cpuid_level: i32 (-1 = no CPUID).
- extended_cpuid_level: u32.
- x86_capability: [u32; NCAPINTS + NBUGINTS] bitmap (CPUID-derived + synthetic + bug flags).
- vmx_capability: [u32; ...] (only if CONFIG_X86_VMX_FEATURE_NAMES).
- x86_vendor_id: [u8; 16] (NUL-terminated "GenuineIntel" / "AuthenticAMD" / ...).
- x86_model_id: [u8; 64] (brand string).
- x86_cache_size: u32.
- loops_per_jiffy: u32.
- cpu_index: u32 (logical CPU number; 0 for boot, set in `identify_secondary_cpu`).
- initialized: bool (set true only after `identify_secondary_cpu` completes).
- topology fields (cpu_die_id, cpu_core_id, ...).

REQ-2: struct cpu_dev:
- c_vendor: &'static str ("Intel" / "AMD" / "Hygon" / "Cyrix" / "Centaur" / "Zhaoxin" / "VIA" / "Transmeta" / "NSC" / ...).
- c_ident: [Option<&'static str>; 2] (e.g. "GenuineIntel" / NULL for Intel).
- c_early_init: Option<fn(&mut CpuInfoX86)> — per-vendor early hook (before full cap harvest).
- c_bsp_init: Option<fn(&mut CpuInfoX86)> — per-boot-only vendor init.
- c_init: Option<fn(&mut CpuInfoX86)> — per-CPU vendor init.
- c_identify: Option<fn(&mut CpuInfoX86)> — per-vendor ident (used when CPUID absent, e.g. Cyrix).
- c_x86_vendor: u8 — the vendor enum stored back into c->x86_vendor.

REQ-3: cpu_devs[] population:
- `init_cpu_devs()` walks linker section `__x86_cpu_dev_start..__x86_cpu_dev_end` (vendor cpu_dev tables registered via `cpu_dev_register`).
- Up to X86_VENDOR_NUM (16) entries.
- Order is registration-time; vendor match is by `c_ident[0]` / `c_ident[1]` string compare against `c->x86_vendor_id`.

REQ-4: early_cpu_init:
- /* Populate cpu_devs[] */
- init_cpu_devs().
- /* Optionally enumerate built-in supported vendors */
- if CONFIG_PROCESSOR_SELECT: pr_info("KERNEL supported cpus:") + iterate.
- /* Boot-CPU ident */
- early_identify_cpu(&boot_cpu_data).

REQ-5: early_identify_cpu(c):
- memset(&c->x86_capability, 0, sizeof).
- c->extended_cpuid_level = 0.
- /* CPUID detection */
- if !cpuid_feature(): identify_cpu_without_cpuid(c).
- if cpuid_feature():
  - cpu_detect(c) — CPUID(0): vendor_id; CPUID(1): family/model/stepping/clflush.
  - get_cpu_vendor(c).
  - intel_unlock_cpuid_leafs(c).
  - get_cpu_cap(c).
  - setup_force_cpu_cap(X86_FEATURE_CPUID).
  - get_cpu_address_sizes(c).
  - cpu_parse_early_param() — cmdline clearcpuid / setcpuid.
  - cpu_init_topology(c).
  - if this_cpu->c_early_init: invoke.
  - c->cpu_index = 0.
  - filter_cpuid_features(c, warn=false).
  - check_cpufeature_deps(c).
  - if this_cpu->c_bsp_init: invoke.
- else: setup_clear_cpu_cap(X86_FEATURE_CPUID); get_cpu_address_sizes(c); cpu_init_topology(c).
- setup_force_cpu_cap(X86_FEATURE_ALWAYS).
- cpu_set_bug_bits(c) — vulnerability matrix (Meltdown / Spectre / MDS / TAA / SRBDS / RFDS / ITS / ...).
- sld_setup(c) — split-lock detect.
- if CONFIG_X86_32: clear PCID + SYSCALL32.
- if !pgtable_l5_enabled(): clear X86_FEATURE_LA57.
- detect_nopl().
- mca_bsp_init(c).

REQ-6: cpu_detect(c):
- cpuid(0x0): cpuid_level + x86_vendor_id (12-byte EBX|EDX|ECX).
- c->x86 = 4 (default).
- if cpuid_level >= 0x1:
  - cpuid(0x1, &tfms, &misc, &junk, &cap0).
  - c->x86 = x86_family(tfms); c->x86_model = x86_model(tfms); c->x86_stepping = x86_stepping(tfms).
  - if cap0 & (1<<19) /* CLFLUSH */:
    - c->x86_clflush_size = ((misc >> 8) & 0xff) * 8.
    - c->x86_cache_alignment = x86_clflush_size.

REQ-7: get_cpu_vendor(c):
- For i in 0..X86_VENDOR_NUM:
  - if !cpu_devs[i]: break.
  - if strcmp(c->x86_vendor_id, cpu_devs[i]->c_ident[0]) == 0
    ∨ (cpu_devs[i]->c_ident[1] ≠ NULL ∧ strcmp(c->x86_vendor_id, cpu_devs[i]->c_ident[1]) == 0):
    - this_cpu = cpu_devs[i].
    - c->x86_vendor = this_cpu->c_x86_vendor.
    - return.
- pr_err_once("CPU: vendor_id '%s' unknown, using generic init."); pr_err("CPU: Your system may be unstable.").
- c->x86_vendor = X86_VENDOR_UNKNOWN.
- this_cpu = &default_cpu.

REQ-8: init_speculation_control(c):
- if cpu_has(c, X86_FEATURE_SPEC_CTRL):
  - set IBRS, IBPB, MSR_SPEC_CTRL.
- if cpu_has(c, X86_FEATURE_INTEL_STIBP): set STIBP.
- if cpu_has(c, X86_FEATURE_SPEC_CTRL_SSBD) ∨ cpu_has(VIRT_SSBD): set SSBD.
- if cpu_has(c, X86_FEATURE_AMD_IBRS): set IBRS + MSR_SPEC_CTRL.
- if cpu_has(c, X86_FEATURE_AMD_IBPB): set IBPB.
- if cpu_has(c, X86_FEATURE_AMD_STIBP): set STIBP + MSR_SPEC_CTRL.
- if cpu_has(c, X86_FEATURE_AMD_SSBD): set SSBD + MSR_SPEC_CTRL + clear VIRT_SSBD.

REQ-9: identify_cpu(c) — full per-CPU identification:
- c->loops_per_jiffy = loops_per_jiffy.
- Reset: x86_cache_size = 0, x86_vendor = UNKNOWN, x86_model = stepping = 0, vendor_id[0] = '\0', model_id[0] = '\0'.
- Per-arch defaults: x86_64: clflush = 64, phys = 36, virt = 48 / x86_32: cpuid_level = -1, clflush = 32, phys = virt = 32.
- x86_cache_alignment = x86_clflush_size.
- memset x86_capability + vmx_capability.
- generic_identify(c).
- cpu_parse_topology(c).
- if this_cpu->c_identify: invoke.
- apply_forced_caps(c) — first pass.
- set_cpu_cap(c, X86_FEATURE_APIC_MSRS_FENCE) — default; AMD/Hygon clear later.
- if this_cpu->c_init: invoke (vendor canonicalization).
- bus_lock_init().
- squash_the_stupid_serial_number(c).
- setup_smep(c); setup_smap(c); setup_umip(c).
- filter_cpuid_features(c, warn=true).
- check_cpufeature_deps(c).
- if !c->x86_model_id[0]:
  - p = table_lookup_model(c).
  - if p: strcpy(model_id, p).
  - else: sprintf(model_id, "%02x/%02x", c->x86, c->x86_model).
- x86_init_rdrand(c).
- setup_pku(c); setup_cet(c).
- apply_forced_caps(c) — second pass.
- if c ≠ &boot_cpu_data:
  - AND boot_cpu_data.x86_capability[0..NCAPINTS] with c->x86_capability.
  - OR (replicate) boot_cpu_data.x86_capability[NCAPINTS..NCAPINTS+NBUGINTS] into c.
- ppin_init(c).
- mcheck_cpu_init(c).
- numa_add_cpu(smp_processor_id()).

REQ-10: identify_boot_cpu / identify_secondary_cpu:
- identify_boot_cpu():
  - identify_cpu(&boot_cpu_data).
  - if HAS_KERNEL_IBT ∧ cpu_feature_enabled(X86_FEATURE_IBT): pr_info("CET detected: Indirect Branch Tracking enabled").
  - if CONFIG_X86_32: enable_sep_cpu().
  - cpu_detect_tlb(&boot_cpu_data).
  - setup_cr_pinning().
  - x86_virt_init(); tsx_init(); tdx_init(); lkgs_init().
- identify_secondary_cpu(cpu):
  - c = &cpu_data(cpu).
  - if !c->initialized: *c = boot_cpu_data /* template */.
  - c->cpu_index = cpu.
  - identify_cpu(c).
  - if CONFIG_X86_32: enable_sep_cpu().
  - x86_spec_ctrl_setup_ap().
  - update_srbds_msr().
  - if boot_cpu_has_bug(X86_BUG_GDS): update_gds_msr().
  - tsx_ap_init().
  - c->initialized = true.

REQ-11: cr4_init / CR pinning:
- cr4_init():
  - cr4 = __read_cr4().
  - if boot_cpu_has(X86_FEATURE_PCID): cr4 |= X86_CR4_PCIDE.
  - if static_branch_likely(&cr_pinning): cr4 = (cr4 & ~cr4_pinned_mask) | cr4_pinned_bits.
  - __write_cr4(cr4).
  - this_cpu_write(cpu_tlbstate.cr4, cr4).
- native_write_cr0(val):
  - asm mov val, %cr0.
  - if cr_pinning ∧ (val & WP) ≠ WP: bits_missing = WP; val |= WP; goto set_register; WARN_ONCE.
- native_write_cr4(val):
  - asm mov val, %cr4.
  - if cr_pinning ∧ (val & cr4_pinned_mask) ≠ cr4_pinned_bits: val = (val & ~cr4_pinned_mask) | cr4_pinned_bits; goto set_register; WARN_ONCE("pinned CR4 bits changed").
- setup_cr_pinning() — boot-only:
  - cr4_pinned_bits = this_cpu_read(cpu_tlbstate.cr4) & cr4_pinned_mask.
  - static_key_enable(&cr_pinning.key).
- cr4_update_irqsoff(set, clear) — lockdep_assert_irqs_disabled:
  - newval = (cr4 & ~clear) | set; if newval ≠ cr4: this_cpu_write + __write_cr4.

REQ-12: GDT layout (`gdt_page`):
- DEFINE_PER_CPU_PAGE_ALIGNED.
- x86_64 entries: KERNEL32_CS, KERNEL_CS (long-mode), KERNEL_DS, DEFAULT_USER32_CS, DEFAULT_USER_DS, DEFAULT_USER_CS.
- x86_32 entries: KERNEL_CS, KERNEL_DS, DEFAULT_USER_CS, DEFAULT_USER_DS, PNPBIOS_*, APMBIOS_*, ESPFIX_SS, PERCPU.
- load_direct_gdt(cpu): load_gdt(get_cpu_gdt_rw(cpu)).
- load_fixmap_gdt(cpu): load_gdt(get_cpu_gdt_ro(cpu)) — read-only mapping via fixmap.
- switch_gdt_and_percpu_base(cpu): load_direct_gdt(cpu); x86_64: wrmsrq(MSR_GS_BASE, cpu_kernelmode_gs_base(cpu)); x86_32: loadsegment(fs, __KERNEL_PERCPU).

REQ-13: TSS / IST setup:
- tss_setup_ist(tss) — x86_64 only:
  - tss->x86_tss.ist[IST_INDEX_DF] = per-CPU DF top.
  - tss->x86_tss.ist[IST_INDEX_NMI] = per-CPU NMI top.
  - tss->x86_tss.ist[IST_INDEX_DB] = per-CPU DB top.
  - tss->x86_tss.ist[IST_INDEX_MCE] = per-CPU MCE top.
  - tss->x86_tss.ist[IST_INDEX_VC] = per-CPU VC top (SEV-ES).
- tss_setup_io_bitmap(tss):
  - io_bitmap_base = IO_BITMAP_OFFSET_INVALID.
  - if CONFIG_X86_IOPL_IOPERM: bitmap = all-0xff (deny), mapall[IO_BITMAP_LONGS] = ~0UL.
- setup_getcpu(cpu): write per-CPU GDT_ENTRY_CPUNODE descriptor encoding vdso_encode_cpunode(cpu, node); if RDTSCP ∨ RDPID: wrmsrq(MSR_TSC_AUX).

REQ-14: cpu_init_exception_handling(boot_cpu):
- tss = this_cpu_ptr(&cpu_tss_rw).
- cpu = raw_smp_processor_id().
- setup_getcpu(cpu).
- if !X86_FEATURE_FRED: tss_setup_ist(tss).
- tss_setup_io_bitmap(tss).
- set_tss_desc(cpu, &get_cpu_entry_area(cpu)->tss.x86_tss).
- load_TR_desc().
- setup_ghcb() — #VC.
- if X86_FEATURE_FSGSBASE: cr4_set_bits(X86_CR4_FSGSBASE); elf_hwcap2 |= HWCAP2_FSGSBASE.
- if X86_FEATURE_FRED: if !boot_cpu: cpu_init_fred_exceptions(); cpu_init_fred_rsps(). else: load_current_idt().

REQ-15: cpu_init() — per-CPU state barrier:
- cur = current.
- cpu = raw_smp_processor_id().
- if CONFIG_NUMA ∧ this_cpu_read(numa_node) == 0 ∧ early_cpu_to_node(cpu) ≠ NUMA_NO_NODE: set_numa_node.
- if X86_64 ∨ VME ∨ TSC ∨ DE: cr4_clear_bits(VME|PVI|TSD|DE).
- if X86_64:
  - loadsegment(fs, 0).
  - memset cur->thread.tls_array.
  - syscall_init().
  - wrmsrq(MSR_FS_BASE, 0); wrmsrq(MSR_KERNEL_GS_BASE, 0); barrier().
  - x2apic_setup().
  - intel_posted_msi_init().
- mmgrab(&init_mm); cur->active_mm = &init_mm; BUG_ON(cur->mm).
- initialize_tlbstate_and_flush(); enter_lazy_tlb(&init_mm, cur).
- load_sp0((unsigned long)(cpu_entry_stack(cpu) + 1)).
- load_mm_ldt(&init_mm).
- initialize_debug_regs(); dbg_restore_debug_regs().
- doublefault_init_cpu_tss().
- if is_uv_system(): uv_cpu_init().
- load_fixmap_gdt(cpu).

REQ-16: arch_cpu_finalize_init:
- c = this_cpu_ptr(&cpu_info).
- identify_boot_cpu().
- select_idle_routine().
- cpu_smt_set_num_threads(__max_threads_per_core, __max_threads_per_core).
- if !CONFIG_SMP: pr_info("CPU: "); print_cpu_info(&boot_cpu_data).
- cpu_select_mitigations().
- arch_smt_update().
- if CONFIG_X86_32 ∧ boot_cpu_data.x86 < 4: panic("Kernel requires i486+ for 'invlpg' and other features").

REQ-17: store_cpu_caps / microcode_check:
- store_cpu_caps(curr_info):
  - curr_info->cpuid_level = cpuid_eax(0).
  - memcpy x86_capability from boot_cpu_data.
  - get_cpu_cap(curr_info).
- microcode_check(prev_info):
  - perf_check_microcode(); amd_check_microcode().
  - store_cpu_caps(&curr_info).
  - if memcmp(prev, curr capability) ≠ 0: pr_warn "CPU features have changed after loading microcode, but might not take effect" + suggest initrd / BIOS update.

REQ-18: Capability override (boot cmdline):
- cpu_caps_cleared[NCAPINTS + NBUGINTS]: bitmap of cleared.
- cpu_caps_set[NCAPINTS + NBUGINTS]: bitmap of set.
- cpu_parse_early_param() parses `clearcpuid=` / `setcpuid=` / `nopcid` / `noinvpcid` / `nofsgsbase` / `nopku` etc.
- apply_forced_caps(c) ANDs/ORs into c->x86_capability twice (before and after vendor c_init).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `boot_cpu_data_initialized_before_use` | INVARIANT | post-`early_identify_cpu`: x86_vendor ≠ uninitialised; vendor_id NUL-terminated. |
| `vendor_id_nul_terminated` | INVARIANT | `get_cpu_vendor`: c.x86_vendor_id is NUL-terminated within [16]. |
| `cpu_devs_bounded` | INVARIANT | `init_cpu_devs`: count ≤ X86_VENDOR_NUM. |
| `apply_forced_caps_bounded` | INVARIANT | loop bound = NCAPINTS + NBUGINTS. |
| `cr4_pinned_bits_subset_of_mask` | INVARIANT | `setup_cr_pinning`: cr4_pinned_bits ⊆ cr4_pinned_mask. |
| `cr4_init_preserves_pinned` | INVARIANT | post-`cr4_init`: (cr4 & cr4_pinned_mask) == cr4_pinned_bits. |
| `write_cr4_pinning_restores` | INVARIANT | post-`native_write_cr4`: when cr_pinning, (read_cr4() & cr4_pinned_mask) == cr4_pinned_bits. |
| `identify_secondary_cpu_idempotent` | INVARIANT | second call with `c.initialized == true` does not memcpy boot_cpu_data. |
| `cpu_index_matches_cpu` | INVARIANT | post-`identify_secondary_cpu(cpu)`: c.cpu_index == cpu. |
| `tss_ist_slots_distinct` | INVARIANT | per-`tss_setup_ist`: DF / NMI / DB / MCE / VC point to distinct per-CPU stacks. |
| `cpu_init_init_mm_grabbed` | INVARIANT | post-`cpu_init`: cur.active_mm == &init_mm; cur.mm is None. |

### Layer 2: TLA+

`arch/x86/cpu-common.tla`:
- States: { uninit, early_ident_done, vendor_dispatched, caps_harvested, full_ident_done, cpu_init_done }.
- Per-CPU init order:
  - `safety_boot_first` — boot CPU completes `early_identify_cpu` before any AP enters `identify_secondary_cpu`.
  - `safety_no_AP_before_init_cpu_devs` — `init_cpu_devs` precedes any vendor dispatch.
  - `safety_AP_clones_only_first_time` — `identify_secondary_cpu(cpu)` only copies `boot_cpu_data` when `!c.initialized`.
  - `safety_initialized_set_last` — `c.initialized = true` is the final write of `identify_secondary_cpu`.
- Capability-AND lattice:
  - `safety_boot_cpu_capability_is_intersection` — after all APs ident, boot_cpu_data.x86_capability[0..NCAPINTS] = ⋂_i AP_i.x86_capability.
  - `safety_bug_bits_replicated` — boot_cpu_data.x86_capability[NCAPINTS..] propagated to every AP.
- Liveness:
  - `liveness_per_AP_eventually_initialized` — every brought-up CPU eventually reaches `c.initialized = true`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `early_cpu_init` post: CPU_DEVS populated ∧ boot_cpu_data has vendor + family + model | `CpuCommon::early_cpu_init` |
| `cpu_detect` post: vendor_id is 12-byte string from CPUID(0) | `CpuCommon::cpu_detect` |
| `get_cpu_vendor` post: c.x86_vendor matches a registered cpu_dev or X86_VENDOR_UNKNOWN | `CpuCommon::get_cpu_vendor` |
| `apply_forced_caps` post: ∀i. c.x86_capability[i] = (input[i] & ~cleared[i]) \| set[i] | `CpuCommon::apply_forced_caps` |
| `init_speculation_control` post: synthetic IBRS/IBPB/STIBP/SSBD bits consistent with vendor primary bits | `CpuCommon::init_speculation_control` |
| `identify_cpu` post: c.x86_capability has bugs replicated from boot_cpu_data; boot_cpu_data has features intersected | `CpuCommon::identify_cpu` |
| `cr4_init` post: cr4_pinned_bits set; cpu_tlbstate.cr4 == read_cr4() | `CpuCommon::cr4_init` |
| `cpu_init_exception_handling` post: TR loaded; tss installed in GDT; FRED-or-IDT path taken exactly once | `CpuCommon::cpu_init_exception_handling` |
| `cpu_init` post: active_mm == init_mm; sp0 == cpu_entry_stack(cpu)+1; init_mm refcount incremented | `CpuCommon::cpu_init` |

### Layer 4: Verus/Creusot functional

`early_cpu_init → init_cpu_devs → early_identify_cpu(boot_cpu) → identify_boot_cpu → setup_cr_pinning → ... → arch_cpu_finalize_init → cpu_select_mitigations → identify_secondary_cpu (per AP) → cpu_init_exception_handling → cpu_init` semantic equivalence to upstream per-`Documentation/arch/x86/boot.rst` + Intel SDM Vol 3A §9.1 (Initialization Overview) + AMD APM Vol 2 §14.1 (Processor Initialization).

### hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

x86 CPU-common reinforcement:

- **Per-CR pinning (WP / SMEP / SMAP / UMIP / FSGSBASE)** — defense against per-ROP CR-clear attack; `native_write_cr0` / `_cr4` re-loop and `WARN_ONCE`.
- **Per-boot setup_cr_pinning snapshots boot CR4** — defense against per-runtime-feature-downgrade.
- **Per-cpu_caps_cleared / cpu_caps_set applied twice** — defense against vendor-`c_init` re-enabling a forbidden feature.
- **Per-X86_VENDOR_UNKNOWN fallback to default_cpu** — defense against unknown-vendor undefined-behavior; `pr_err_once` audit trail.
- **Per-NCAPINTS+NBUGINTS bounded capability loops** — defense against per-OOB write on cap bitmap.
- **Per-c.initialized guard in identify_secondary_cpu** — defense against per-AP capability-erasure on resume / re-online.
- **Per-AND capability intersection in identify_cpu** — defense against per-heterogeneous CPU feature drift (only common features advertised).
- **Per-OR bug-bit replication** — defense against per-vulnerability silent-mismatch across CPUs.
- **Per-init_speculation_control synthetic flags** — defense against per-mismatched-vendor IBRS/STIBP enablement.
- **Per-TSS IST per-CPU stack ownership** — defense against per-cross-CPU exception-stack sharing.
- **Per-load_TR_desc + set_tss_desc** — defense against per-stale TR pointing at freed TSS.
- **Per-cpu_init mmgrab(init_mm) + active_mm assignment** — defense against per-CPU bring-up TLB-poison.
- **Per-microcode_check feature-drift diagnostic** — defense against per-microcode-load silent capability disappearance.

