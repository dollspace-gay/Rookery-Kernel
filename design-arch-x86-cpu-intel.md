---
title: "Tier-3: arch/x86/kernel/cpu/intel.c — Intel CPU init, errata and feature gating"
tags: ["tier-3", "arch", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

Per-Intel-vendor cpu_dev provides early + per-CPU + BSP-only init for Intel processors and patches up `struct cpuinfo_x86` to reflect family/model-specific behaviour. Per-`early_init_intel(c)` runs before global CPU feature enumeration and clears mis-reported features (Spectre-broken microcode, ATOM PSE erratum, P4 phys-bits, Yonah PAT erratum, Quark PGE). Per-`init_intel(c)` runs after enumeration and adds DS/BTS/PEBS, CLFLUSH/MONITOR errata, ZMM-downclock workaround (PREFER_YMM), zenith TLB topology, fast-string MSR (`MSR_IA32_MISC_ENABLE.FAST_STRING`), TME phys-bits adjustment, microcode-features-shadow (`MSR_MISC_FEATURES_ENABLES`), CPUID-FAULT enablement (`MSR_PLATFORM_INFO`), thermal init and split-lock init. Per-`bsp_init_intel(c)` runs once on the BSP to detect resctrl. Per-`intel_detect_tlb` uses CPUID leaf 0x2 (legacy descriptor table) to populate `tlb_lli_*`/`tlb_lld_*`. Per-`intel_unlock_cpuid_leafs` clears `MSR_IA32_MISC_ENABLE.LIMIT_CPUID` when BIOS limited CPUID to leaf 2. Per-`probe_xeon_phi_r3mwait` enables Ring-3 MONITOR/MWAIT only on Knights-Landing/Mill. Critical for: correct Intel-microarchitecture quirks, dependable feature gating, defense against broken-BIOS / broken-microcode states.

This Tier-3 covers `arch/x86/kernel/cpu/intel.c` (~786 lines).

### Acceptance Criteria

- [ ] AC-1: Intel vendor cpu_dev is link-registered; cpuid "GenuineIntel" routes to early_init_intel + bsp_init_intel + init_intel + intel_detect_tlb.
- [ ] AC-2: early_init_intel sets CONSTANT_TSC when 8000_0007.EDX[8] or vfm in whitelist; otherwise leaves it clear.
- [ ] AC-3: bad_spectre_microcode matches vfm+stepping in table ∧ microcode<=table.microcode ∧ !HYPERVISOR: spectre caps cleared.
- [ ] AC-4: ATOM_BONNELL stepping<=2 microcode<0x20e: X86_FEATURE_PSE cleared.
- [ ] AC-5: vfm == QUARK_X1000: X86_FEATURE_PGE cleared.
- [ ] AC-6: vfm in {YONAH..earlier-than-CORE-Yonah PAT range INTEL_PENTIUM_PRO..INTEL_CORE_YONAH}: X86_FEATURE_PAT cleared.
- [ ] AC-7: MSR_IA32_MISC_ENABLE.FAST_STRING absent: REP_GOOD ∧ ERMS cleared, log "Disabled fast string operations".
- [ ] AC-8: detect_tme_early on locked+enabled TME: x86_phys_bits decreased by KEYID_BITS; if not locked/enabled: TME cap cleared.
- [ ] AC-9: probe_xeon_phi_r3mwait sets RING3MWAIT and HWCAP2_RING3MWAIT for boot CPU on Knights-Landing/Mill unless ring3mwait=disable.
- [ ] AC-10: init_intel sets PEBS/BTS based on MSR_IA32_MISC_ENABLE bits; sets ARCH_PERFMON when CPUID.10 reports >1 counter.
- [ ] AC-11: x86_match_cpu(zmm_exclusion_list) ⟹ PREFER_YMM set; CLFLUSH_MONITOR set on Dunnington/Nehalem-EX/Westmere-EX; MONITOR set on Goldmont/Lunar-Lake-M.
- [ ] AC-12: intel_unlock_cpuid_leafs clears MSR_IA32_MISC_ENABLE.LIMIT_CPUID and refreshes c.cpuid_level.
- [ ] AC-13: intel_detect_tlb reads CPUID leaf 0x2 only when cpuid_level>=2 and updates tlb_lli/lld_* via intel_tlb_lookup (max-merge semantics).
- [ ] AC-14: bsp_init_intel invokes resctrl_cpu_detect on the boot CPU.
- [ ] AC-15: forcepae=1: PAE forced + TAINT_CPU_OUT_OF_SPEC raised.

### Architecture

```
struct IntelCpuDev {
  vendor: &'static str,                // "Intel"
  ident: &'static [&'static str],      // { "GenuineIntel" }
  x86_vendor: u8,                      // X86_VENDOR_INTEL
  early_init: fn(&mut CpuInfoX86),     // Intel::early_init
  bsp_init:   fn(&mut CpuInfoX86),     // Intel::bsp_init
  init:       fn(&mut CpuInfoX86),     // Intel::init
  detect_tlb: fn(&mut CpuInfoX86),     // Intel::detect_tlb
  legacy_models:     Option<LegacyModels>,    // X86_32 only
  legacy_cache_size: Option<fn(&CpuInfoX86, u32) -> u32>, // X86_32 only
}

struct SkuMicrocode { vfm: u32, stepping: u8, microcode: u32 }

static SPECTRE_BAD_MICROCODES: &[SkuMicrocode] = &[
  /* Kabylake, Skylake-X, Broadwell variants, Haswell variants, Ivybridge-X,
     Sandybridge-X — see intel.c spectre_bad_microcodes[] */
];

static ZMM_EXCLUSION_LIST: &[X86CpuId] = &[
  /* Skylake-X, Icelake variants, Tigerlake variants */
];
```

`Intel::early_init(c)`:
1. /* Microcode revision latch */
2. if c.x86 >= 6 ∧ !cpu_has(c, X86_FEATURE_IA64): c.microcode = intel_get_microcode_revision().
3. c.intel_platform_id = intel_get_platform_id().
4. /* Spectre microcode blacklist */
5. if any-of(SPEC_CTRL, INTEL_STIBP, IBRS, IBPB, STIBP) ∧ Intel::bad_spectre_microcode(c):
   - clear_caps([IBRS, IBPB, STIBP, SPEC_CTRL, MSR_SPEC_CTRL, INTEL_STIBP, SSBD, SPEC_CTRL_SSBD]).
6. /* Atom Bonnell AAE44 */
7. if c.x86_vfm == INTEL_ATOM_BONNELL ∧ c.x86_stepping <= 2 ∧ c.microcode < 0x20e:
   - clear_cpu_cap(c, PSE).
8. /* P4 Prescott phys_bits / Netburst cache-alignment */
9. if !X86_64 ∧ c.x86 == 15 ∧ c.x86_cache_alignment == 64: c.x86_cache_alignment = 128.
10. if c.x86_vfm == INTEL_P4_PRESCOTT ∧ (stepping ∈ {3, 4}): c.x86_phys_bits = 36.
11. /* Constant/non-stop TSC */
12. if c.x86_power & (1 << 8): set CONSTANT_TSC ∧ NONSTOP_TSC.
13. else if vfm ∈ Prescott..Cedarmill ∨ vfm ∈ Yonah..Ivybridge: set CONSTANT_TSC.
14. /* Atom NONSTOP_TSC_S3 */
15. match c.x86_vfm in {SALTWELL_MID, SALTWELL_TABLET, SILVERMONT_MID, AIRMONT_NP}: set NONSTOP_TSC_S3.
16. /* Yonah PAT erratum */
17. if PENTIUM_PRO <= vfm <= CORE_YONAH: clear PAT.
18. /* Fast-string MSR */
19. if vfm >= INTEL_PENTIUM_M_DOTHAN:
    - rdmsrq(MSR_IA32_MISC_ENABLE, &misc).
    - if misc & FAST_STRING: set REP_GOOD.
    - else: clear REP_GOOD ∧ ERMS; pr_info("Disabled fast string operations").
20. /* Quark X1000 PGE */
21. if vfm == QUARK_X1000: clear PGE.
22. Intel::check_selfsnoop_errata(c).
23. /* TME phys-bits */
24. if cpu_has(c, TME): Intel::detect_tme_early(c).

`Intel::bsp_init(c)`:
1. resctrl_cpu_detect(c).

`Intel::init(c)`:
1. Intel::early_init(c).
2. Intel::workarounds(c).
3. init_intel_cacheinfo(c).
4. /* Arch PerfMon */
5. if c.cpuid_level > 9:
   - eax = cpuid_eax(10).
   - if (eax & 0xff) ∧ ((eax >> 8) & 0xff) > 1: set ARCH_PERFMON.
6. if cpu_has(c, XMM2): set LFENCE_RDTSC.
7. /* DS / BTS / PEBS */
8. if boot_cpu_has(DS):
   - rdmsr(MSR_IA32_MISC_ENABLE, l1, l2).
   - if !(l1 & BTS_UNAVAIL): set BTS.
   - if !(l1 & PEBS_UNAVAIL): set PEBS.
9. /* CLFLUSH+MONITOR / MONITOR errata */
10. if boot_cpu_has(CLFLUSH) ∧ vfm ∈ {CORE2_DUNNINGTON, NEHALEM_EX, WESTMERE_EX}: set X86_BUG_CLFLUSH_MONITOR.
11. if boot_cpu_has(MWAIT) ∧ vfm ∈ {ATOM_GOLDMONT, LUNARLAKE_M}: set X86_BUG_MONITOR.
12. /* P4 cache-alignment doubling (X86_64) */
13. if X86_64 ∧ c.x86 == 15: c.x86_cache_alignment = c.x86_clflush_size * 2.
14. /* Legacy PII/PIII model-name disambiguation (X86_32) */
15. if !X86_64 ∧ c.x86 == 6: switch (x86_model, l2): Celeron-A / Mendocino / Coppermine / Dixon / Covington overrides.
16. if x86_match_cpu(ZMM_EXCLUSION_LIST): set PREFER_YMM.
17. srat_detect_node(c).
18. init_ia32_feat_ctl(c).
19. Intel::init_misc_features(c).
20. split_lock_init().
21. intel_init_thermal(c).

`Intel::workarounds(c)` [X86_32]:
1. /* F00F */
2. if INTEL_FAM5_START <= vfm < INTEL_QUARK_X1000: set X86_BUG_F00F.
3. /* SEP / klamath stepping */
4. if (vfm == KLAMATH ∧ stepping < 3) ∨ vfm < KLAMATH: clear SEP.
5. /* forcepae taint */
6. if forcepae: set PAE; add_taint(TAINT_CPU_OUT_OF_SPEC, LOCKDEP_NOW_UNRELIABLE).
7. /* P4 Willamette HW-prefetch disable */
8. if vfm == P4_WILLAMETTE ∧ stepping == 1: msr_set_bit(MSR_IA32_MISC_ENABLE, PREFETCH_DISABLE_BIT).
9. /* 11AP P54C */
10. if boot_cpu_has(APIC) ∧ vfm == PENTIUM_75 ∧ (stepping < 6 ∨ stepping == 0xb): set X86_BUG_11AP.
11. /* MOVSL alignment preference */
12. if vfm >= PENTIUM_PRO: movsl_mask.mask = 7.
13. Intel::smp_check(c).

`Intel::detect_tme_early(c)`:
1. rdmsrq(MSR_IA32_TME_ACTIVATE, &tme).
2. if !LOCKED(tme) ∨ !ENABLED(tme):
   - pr_info_once("x86/tme: not enabled by BIOS"); clear_cpu_cap(c, TME); return.
3. pr_info_once("x86/tme: enabled by BIOS").
4. keyid_bits = (tme >> 32) & 0xf.
5. if keyid_bits == 0: return.
6. c.x86_phys_bits -= keyid_bits.
7. pr_info_once("x86/mktme: BIOS enabled: x86_phys_bits reduced by %d", keyid_bits).

`Intel::init_misc_features(c)`:
1. if rdmsrq_safe(MSR_MISC_FEATURES_ENABLES, &msr) != 0: return.
2. this_cpu_write(msr_misc_features_shadow, 0).
3. Intel::init_cpuid_fault(c).
4. Intel::probe_xeon_phi_r3mwait(c).
5. msr = this_cpu_read(msr_misc_features_shadow).
6. wrmsrq(MSR_MISC_FEATURES_ENABLES, msr).

`Intel::detect_tlb(c)`:
1. if c.cpuid_level < 2: return.
2. cpuid_leaf_0x2(&regs).
3. for_each_cpuid_0x2_desc(regs, ptr, desc): Intel::tlb_lookup(desc).

### Out of Scope

- arch/x86/kernel/cpu/common.c (covered in `cpu-common.md` Tier-3)
- arch/x86/kernel/cpu/bugs.c / mitigations (covered in `cpu-mitigations.md` Tier-3)
- arch/x86/kernel/cpu/cacheinfo.c (covered separately if expanded)
- arch/x86/kernel/cpu/resctrl/* (covered separately if expanded)
- arch/x86/events/intel/* PMU drivers (covered separately if expanded)
- arch/x86/kernel/tsc.c / TSC calibration (covered in `tsc.md` Tier-3)
- split_lock_detect.c (covered separately if expanded)
- intel_init_thermal / arch/x86/kernel/cpu/intel_epb.c (covered separately if expanded)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cpu_dev intel_cpu_dev` | per-vendor dispatch | `IntelCpuDev` |
| `cpu_dev_register(intel_cpu_dev)` | per-link-time registration | `IntelCpuDev::register` |
| `early_init_intel(c)` | per-CPU pre-enum init | `Intel::early_init` |
| `init_intel(c)` | per-CPU post-enum init | `Intel::init` |
| `bsp_init_intel(c)` | per-BSP-only init | `Intel::bsp_init` |
| `intel_workarounds(c)` | per-CPU 32-bit errata | `Intel::workarounds` |
| `intel_smp_check(c)` | per-secondary SMP-stepping check | `Intel::smp_check` |
| `intel_detect_tlb(c)` | per-CPU TLB enumeration | `Intel::detect_tlb` |
| `intel_tlb_lookup(desc)` | per-leaf-0x2-descriptor decoder | `Intel::tlb_lookup` |
| `intel_unlock_cpuid_leafs(c)` | per-CPU CPUID-limit unlock | `Intel::unlock_cpuid_leafs` |
| `intel_size_cache(c, size)` | per-CPU legacy cache-size override | `Intel::size_cache` |
| `bad_spectre_microcode(c)` | per-CPU broken-ucode predicate | `Intel::bad_spectre_microcode` |
| `check_memory_type_self_snoop_errata(c)` | per-CPU SELFSNOOP erratum | `Intel::check_selfsnoop_errata` |
| `probe_xeon_phi_r3mwait(c)` | per-CPU R3MWAIT probe | `Intel::probe_xeon_phi_r3mwait` |
| `detect_tme_early(c)` | per-CPU TME/MKTME phys-bit fixup | `Intel::detect_tme_early` |
| `init_intel_misc_features(c)` | per-CPU MISC_FEATURES_ENABLES | `Intel::init_misc_features` |
| `init_cpuid_fault(c)` | per-CPU CPUID-FAULT detect | `Intel::init_cpuid_fault` |
| `ppro_with_ram_bug()` | per-i686 PPro errata #50 | `Intel::ppro_with_ram_bug` |
| `spectre_bad_microcodes[]` | per-table broken-ucode list | `Intel::SPECTRE_BAD_MICROCODES` |
| `zmm_exclusion_list[]` | per-table ZMM-downclock list | `Intel::ZMM_EXCLUSION_LIST` |
| `ring3mwait_disabled` | per-cmdline flag | `Intel::ring3mwait_disabled` |
| `forcepae` | per-cmdline flag | `Intel::forcepae` |
| `msr_misc_features_shadow` | per-CPU MSR shadow | `Intel::misc_features_shadow` |

### compatibility contract

REQ-1: struct intel_cpu_dev:
- c_vendor = "Intel".
- c_ident = { "GenuineIntel" }.
- c_x86_vendor = X86_VENDOR_INTEL.
- c_early_init = early_init_intel.
- c_bsp_init = bsp_init_intel.
- c_init = init_intel.
- c_detect_tlb = intel_detect_tlb.
- legacy_models / legacy_cache_size: per-CONFIG_X86_32 only.

REQ-2: early_init_intel(c):
- /* Family >= 6, !IA64: latch microcode revision */
- if c.x86 >= 6 ∧ !cpu_has(c, X86_FEATURE_IA64): c.microcode = intel_get_microcode_revision().
- c.intel_platform_id = intel_get_platform_id().
- /* Spectre-broken ucode blacklist */
- if (SPEC_CTRL ∨ INTEL_STIBP ∨ IBRS ∨ IBPB ∨ STIBP) ∧ bad_spectre_microcode(c):
  - setup_clear_cpu_cap of IBRS, IBPB, STIBP, SPEC_CTRL, MSR_SPEC_CTRL, INTEL_STIBP, SSBD, SPEC_CTRL_SSBD.
- /* Atom Bonnell AAE44 etc. */
- if c.x86_vfm == INTEL_ATOM_BONNELL ∧ c.x86_stepping <= 2 ∧ c.microcode < 0x20e:
  - clear_cpu_cap(c, X86_FEATURE_PSE).
- /* P4 Netburst clflush bytes-vs-IO mismatch (X86_32 only) */
- if !CONFIG_X86_64 ∧ c.x86 == 15 ∧ c.x86_cache_alignment == 64: c.x86_cache_alignment = 128.
- /* P4 Prescott 0F33/0F34 phys_bits=36 */
- if c.x86_vfm == INTEL_P4_PRESCOTT ∧ (stepping == 3 ∨ stepping == 4): c.x86_phys_bits = 36.
- /* Constant/non-stop TSC from 8000_0007 EDX bit 8 + model whitelist */
- if c.x86_power & (1 << 8): set CONSTANT_TSC + NONSTOP_TSC.
- else if INTEL_P4_PRESCOTT..INTEL_P4_CEDARMILL ∨ INTEL_CORE_YONAH..INTEL_IVYBRIDGE: set CONSTANT_TSC.
- /* Atom S3-NONSTOP_TSC */
- if vfm ∈ {ATOM_SALTWELL_MID, ATOM_SALTWELL_TABLET, ATOM_SILVERMONT_MID, ATOM_AIRMONT_NP}: set NONSTOP_TSC_S3.
- /* Yonah AN7 PAT erratum */
- if INTEL_PENTIUM_PRO <= vfm <= INTEL_CORE_YONAH: clear X86_FEATURE_PAT.
- /* MISC_ENABLE.FAST_STRING */
- if vfm >= INTEL_PENTIUM_M_DOTHAN:
  - rdmsrq(MSR_IA32_MISC_ENABLE, misc_enable).
  - if misc_enable & FAST_STRING: set X86_FEATURE_REP_GOOD.
  - else: clear REP_GOOD ∧ ERMS, pr_info("Disabled fast string operations").
- /* Quark X1000 PGE erratum */
- if vfm == INTEL_QUARK_X1000: clear X86_FEATURE_PGE.
- check_memory_type_self_snoop_errata(c).
- /* TME phys-bits adjustment */
- if cpu_has(c, X86_FEATURE_TME): detect_tme_early(c).

REQ-3: intel_unlock_cpuid_leafs(c):
- if boot_cpu_data.x86_vendor != X86_VENDOR_INTEL: return.
- if c.x86_vfm < INTEL_PENTIUM_M_DOTHAN: return.
- /* Clear MSR_IA32_MISC_ENABLE.LIMIT_CPUID and re-read max leaf */
- if msr_clear_bit(MSR_IA32_MISC_ENABLE, MSR_IA32_MISC_ENABLE_LIMIT_CPUID_BIT) > 0:
  - c.cpuid_level = cpuid_eax(0).

REQ-4: bsp_init_intel(c):
- resctrl_cpu_detect(c).

REQ-5: init_intel(c):
- early_init_intel(c).
- intel_workarounds(c).
- init_intel_cacheinfo(c).
- /* PMU detect */
- if c.cpuid_level > 9:
  - eax = cpuid_eax(10).
  - if (eax & 0xff) ∧ ((eax >> 8) & 0xff) > 1: set X86_FEATURE_ARCH_PERFMON.
- /* LFENCE-RDTSC */
- if cpu_has(c, X86_FEATURE_XMM2): set LFENCE_RDTSC.
- /* DS / BTS / PEBS via MSR_IA32_MISC_ENABLE */
- if boot_cpu_has(X86_FEATURE_DS):
  - rdmsr(MSR_IA32_MISC_ENABLE, l1, l2).
  - if !(l1 & MSR_IA32_MISC_ENABLE_BTS_UNAVAIL): set BTS.
  - if !(l1 & MSR_IA32_MISC_ENABLE_PEBS_UNAVAIL): set PEBS.
- /* CLFLUSH+MONITOR errata */
- if boot_cpu_has(CLFLUSH) ∧ vfm ∈ {CORE2_DUNNINGTON, NEHALEM_EX, WESTMERE_EX}: set X86_BUG_CLFLUSH_MONITOR.
- /* MONITOR errata */
- if boot_cpu_has(MWAIT) ∧ vfm ∈ {ATOM_GOLDMONT, LUNARLAKE_M}: set X86_BUG_MONITOR.
- /* P4 family-15 cache-alignment doubling (X86_64) */
- if CONFIG_X86_64 ∧ c.x86 == 15: c.x86_cache_alignment = c.x86_clflush_size * 2.
- /* PII/PIII Celeron model-name disambiguation by L2 size (X86_32) */
- if !CONFIG_X86_64 ∧ c.x86 == 6: classify by (x86_model, x86_cache_size, x86_stepping) into "Celeron (Covington/Mendocino/Coppermine)", "Mobile Pentium II (Dixon)", "Celeron-A".
- /* ZMM downclock list ⟹ prefer YMM */
- if x86_match_cpu(zmm_exclusion_list): set X86_FEATURE_PREFER_YMM.
- srat_detect_node(c).
- init_ia32_feat_ctl(c).
- init_intel_misc_features(c).
- split_lock_init().
- intel_init_thermal(c).

REQ-6: intel_workarounds(c) [X86_32 only]:
- /* F00F bug: Pentium..QUARK_X1000 */
- if CONFIG_X86_F00F_BUG ∧ INTEL_FAM5_START <= vfm < INTEL_QUARK_X1000: set X86_BUG_F00F.
- /* SEP cap clear on PPro / early Klamath */
- if (vfm == INTEL_PENTIUM_II_KLAMATH ∧ stepping < 3) ∨ vfm < INTEL_PENTIUM_II_KLAMATH: clear X86_FEATURE_SEP.
- /* forcepae cmdline ⟹ set X86_FEATURE_PAE + taint */
- if forcepae: set PAE ∧ add_taint(TAINT_CPU_OUT_OF_SPEC, LOCKDEP_NOW_UNRELIABLE).
- /* P4 Willamette stepping-1 erratum 037: disable HW prefetch */
- if vfm == INTEL_P4_WILLAMETTE ∧ stepping == 1: msr_set_bit(MSR_IA32_MISC_ENABLE, PREFETCH_DISABLE_BIT).
- /* P54C 11AP integrated-APIC erratum */
- if boot_cpu_has(APIC) ∧ vfm == INTEL_PENTIUM_75 ∧ (stepping < 6 ∨ stepping == 0xb): set X86_BUG_11AP.
- /* PII/PIII MOVSL alignment preference */
- if CONFIG_X86_INTEL_USERCOPY ∧ vfm >= INTEL_PENTIUM_PRO: movsl_mask.mask = 7.
- intel_smp_check(c).

REQ-7: intel_smp_check(c) [X86_32 secondary]:
- if !c.cpu_index: return.
- if INTEL_FAM5_START <= vfm < INTEL_PENTIUM_MMX ∧ 1 <= stepping <= 4: WARN_ONCE("WARNING: SMP operation may be unreliable with B stepping processors").

REQ-8: check_memory_type_self_snoop_errata(c):
- if vfm ∈ {CORE_YONAH, CORE2_{MEROM,MEROM_L,PENRYN,DUNNINGTON}, NEHALEM, NEHALEM_G, NEHALEM_EP, NEHALEM_EX, WESTMERE, WESTMERE_EP, SANDYBRIDGE}: setup_clear_cpu_cap(X86_FEATURE_SELFSNOOP).

REQ-9: bad_spectre_microcode(c):
- /* Hypervisors may lie about microcode revision */
- if cpu_has(c, X86_FEATURE_HYPERVISOR): return false.
- for entry in SPECTRE_BAD_MICROCODES:
  - if vfm == entry.vfm ∧ stepping == entry.stepping: return c.microcode <= entry.microcode.
- return false.

REQ-10: probe_xeon_phi_r3mwait(c):
- if c.x86 != 6: return.
- if vfm ∉ {XEON_PHI_KNL, XEON_PHI_KNM}: return.
- if ring3mwait_disabled: return.
- set X86_FEATURE_RING3MWAIT.
- this_cpu_or(msr_misc_features_shadow, 1 << MSR_MISC_FEATURES_ENABLES_RING3MWAIT_BIT).
- if c == &boot_cpu_data: ELF_HWCAP2 |= HWCAP2_RING3MWAIT.

REQ-11: detect_tme_early(c):
- rdmsrq(MSR_IA32_TME_ACTIVATE, tme_activate).
- if !TME_ACTIVATE_LOCKED(tme_activate) ∨ !TME_ACTIVATE_ENABLED(tme_activate):
  - pr_info_once("x86/tme: not enabled by BIOS").
  - clear X86_FEATURE_TME.
  - return.
- pr_info_once("x86/tme: enabled by BIOS").
- keyid_bits = TME_ACTIVATE_KEYID_BITS(tme_activate) /* bits 35:32 */.
- if !keyid_bits: return.
- /* KeyID bits effectively reduce physical address bits */
- c.x86_phys_bits -= keyid_bits.
- pr_info_once("x86/mktme: BIOS enabled: x86_phys_bits reduced by %d", keyid_bits).

REQ-12: init_intel_misc_features(c):
- if rdmsrq_safe(MSR_MISC_FEATURES_ENABLES, &msr) != 0: return.
- /* Clear shadow */
- this_cpu_write(msr_misc_features_shadow, 0).
- init_cpuid_fault(c).
- probe_xeon_phi_r3mwait(c).
- msr = this_cpu_read(msr_misc_features_shadow).
- wrmsrq(MSR_MISC_FEATURES_ENABLES, msr).

REQ-13: init_cpuid_fault(c):
- if rdmsrq_safe(MSR_PLATFORM_INFO, &msr) == 0:
  - if msr & MSR_PLATFORM_INFO_CPUID_FAULT: set X86_FEATURE_CPUID_FAULT.

REQ-14: intel_detect_tlb(c):
- if c.cpuid_level < 2: return.
- cpuid_leaf_0x2(&regs).
- for_each_cpuid_0x2_desc(regs, ptr, desc): intel_tlb_lookup(desc).

REQ-15: intel_tlb_lookup(desc):
- /* Max-merge per descriptor type */
- STLB_4K ⟹ update tlb_lli_4k, tlb_lld_4k.
- STLB_4K_2M ⟹ update tlb_lli_4k, tlb_lld_4k, tlb_lli_2m, tlb_lld_2m, tlb_lli_4m, tlb_lld_4m.
- TLB_INST_ALL ⟹ tlb_lli_4k / tlb_lli_2m / tlb_lli_4m.
- TLB_INST_4K / 4M / 2M_4M ⟹ corresponding tlb_lli_*.
- TLB_DATA_*4K / 4M / 2M_4M / 4K_4M ⟹ corresponding tlb_lld_*.
- TLB_DATA_1G_2M_4M ⟹ tlb_lld_2m / 4m bumped to TLB_0x63_2M_4M_ENTRIES, then fallthrough.
- TLB_DATA_1G ⟹ tlb_lld_1g.

REQ-16: intel_size_cache(c, size) [X86_32]:
- if vfm == INTEL_PENTIUM_III_TUALATIN ∧ size == 0: size = 256.
- if vfm == INTEL_QUARK_X1000: size = 16.
- return size.

REQ-17: ppro_with_ram_bug() [X86_32]:
- if boot_cpu_data.x86_vfm == INTEL_PENTIUM_PRO ∧ boot_cpu_data.x86_stepping < 8:
  - pr_info("Pentium Pro with Errata#50 detected. Taking evasive action.").
  - return 1.
- return 0.

REQ-18: Per-cmdline `ring3mwait=disable` ⟹ ring3mwait_disabled = true. Per-cmdline `forcepae` ⟹ forcepae = 1.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `spectre_microcode_table_sorted_no_uaf` | INVARIANT | per-bad_spectre_microcode: table iteration in-bounds, no UAF. |
| `tme_keyid_bits_within_phys` | INVARIANT | per-detect_tme_early: keyid_bits ≤ c.x86_phys_bits before subtract. |
| `misc_features_shadow_consistent` | INVARIANT | per-init_intel_misc_features: shadow written == MSR written. |
| `tlb_lookup_max_merge_monotone` | INVARIANT | per-intel_tlb_lookup: tlb_lli/lld_* only ever increase. |
| `quark_pge_cleared` | INVARIANT | per-early_init_intel: vfm==QUARK_X1000 ⟹ PGE cleared post-call. |
| `bonnell_pse_cleared_when_stale_ucode` | INVARIANT | per-early_init_intel: ATOM_BONNELL ∧ stepping<=2 ∧ ucode<0x20e ⟹ PSE cleared. |
| `pat_disabled_on_yonah_range` | INVARIANT | per-early_init_intel: PENTIUM_PRO..CORE_YONAH ⟹ PAT cleared. |

### Layer 2: TLA+

`arch/x86/cpu-intel.tla`:
- Per-CPU-init-pipeline: pre-enum (early_init) → enum → post-enum (init) → bsp-only (bsp_init).
- Properties:
  - `safety_no_spectre_caps_when_blacklisted` — per-bad_spectre_microcode == true ⟹ post-early_init, all spectre caps cleared.
  - `safety_quark_pge_cleared` — per-vfm == QUARK_X1000 ⟹ post-early_init, PGE clear.
  - `safety_atom_pse_cleared_when_stale_ucode` — per-ATOM_BONNELL ∧ stepping ≤ 2 ∧ ucode < 0x20e ⟹ post-early_init, PSE clear.
  - `safety_fast_string_implies_rep_good` — per-MSR_IA32_MISC_ENABLE.FAST_STRING set ⟹ post-early_init, REP_GOOD set.
  - `safety_zmm_exclusion_implies_prefer_ymm` — per-x86_match_cpu(zmm_exclusion_list) ⟹ post-init, PREFER_YMM set.
  - `safety_tme_phys_bits_monotone_decrease` — per-detect_tme_early: x86_phys_bits decreases by exactly keyid_bits.
  - `liveness_per_CPU_init_terminates` — per-init_intel: returns in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Intel::early_init` post: spectre caps cleared iff bad_spectre_microcode | `Intel::early_init` |
| `Intel::early_init` post: TME cap cleared iff !LOCKED ∨ !ENABLED | `Intel::detect_tme_early` |
| `Intel::init` post: ARCH_PERFMON iff CPUID.10 reports >1 counter | `Intel::init` |
| `Intel::init` post: BTS / PEBS exactly mirror MSR_IA32_MISC_ENABLE bits | `Intel::init` |
| `Intel::detect_tlb` post: tlb_lli/lld_* are max over all descriptors seen | `Intel::detect_tlb` |
| `Intel::bsp_init` post: resctrl_cpu_detect called exactly once on BSP | `Intel::bsp_init` |
| `Intel::unlock_cpuid_leafs` post: LIMIT_CPUID == 0; cpuid_level refreshed | `Intel::unlock_cpuid_leafs` |

### Layer 4: Verus/Creusot functional

`Per-Intel-CPU-init: early_init_intel → identify_cpu (common) → init_intel → bsp_init_intel (BSP only) → intel_detect_tlb` semantic equivalence: per-SDM Vol 3A § "Processor Identification and Feature Determination" and Vol 4 (MSRs MSR_IA32_MISC_ENABLE, MSR_MISC_FEATURES_ENABLES, MSR_PLATFORM_INFO, MSR_IA32_TME_ACTIVATE).

### hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening and `cpu-common.md`.)

Intel-cpu reinforcement:

- **Per-bad-Spectre-microcode blacklist** — defense against per-stale-ucode false-claim of IBRS/IBPB/STIBP/SPEC_CTRL/SSBD.
- **Per-HYPERVISOR-aware microcode-version check** — defense against per-hypervisor-lying ucode-version trapping kernel into wrong policy.
- **Per-ATOM_BONNELL PSE clear on stale ucode** — defense against per-PSE-speculation race (AAE44/AAF40/AAG38/AAH41).
- **Per-Quark X1000 PGE clear** — defense against per-stale-TLB (Quark DevMan_001 § 6.4.11).
- **Per-Yonah-range PAT clear** — defense against per-AN7-USWC-consolidation to UC.
- **Per-TME LOCKED∧ENABLED gating + KeyID-bits phys_bits decrement** — defense against per-MKTME aliasing (physical-address-aliased keys).
- **Per-MSR_IA32_MISC_ENABLE.LIMIT_CPUID unlock guarded by vfm >= Dothan** — defense against per-mis-CPUID early-CPU enumeration.
- **Per-MSR_MISC_FEATURES_ENABLES shadow strictly drives wrmsrq** — defense against per-uninitialized-shadow racing with feature gates.
- **Per-CPUID-FAULT autodetect via MSR_PLATFORM_INFO** — defense against unprivileged CPUID-side-channel.
- **Per-ZMM-downclock exclusion-list ⟹ PREFER_YMM** — defense against per-AVX-512-frequency-collapse DoS on shared CPUs.
- **Per-X86_BUG_CLFLUSH_MONITOR / X86_BUG_MONITOR errata bits set** — defense against per-erratum-induced wake-up loss.
- **Per-forcepae taint TAINT_CPU_OUT_OF_SPEC + LOCKDEP_NOW_UNRELIABLE** — defense against silently-running unsupported PAE.
- **Per-intel_unlock_cpuid_leafs vendor-guarded** — defense against per-cross-vendor misuse on AMD.

