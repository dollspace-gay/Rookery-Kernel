---
title: "Tier-3: arch/x86/kernel/cpu/amd.c — AMD CPU init, errata and feature gating"
tags: ["tier-3", "arch", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

Per-AMD-vendor cpu_dev provides early + per-CPU + BSP-only init for AMD processors and patches up `struct cpuinfo_x86` for family/model-specific errata and feature gating. Per-`early_init_amd(c)` sets K8 cap for family >= 0xf, latches microcode revision from `MSR_AMD64_PATCH_LEVEL`, sets CONSTANT_TSC / NONSTOP_TSC / ACC_POWER / RAPL from `c->x86_power` (8000_0007 EDX bits), applies F16h erratum 793 (LS_CFG bit 15) and IBPB_BRTYPE / SBPB synthesis. Per-`init_amd(c)` dispatches per-family fixups (`init_amd_k5/k6/k7/k8/gh/ln/bd/jg`) plus per-Zen-generation fixups (`init_amd_zen1..5`), handles SVM-disabled-in-BIOS (`MSR_VM_CR.SVM_DIS`), LFENCE-serializing via `MSR_AMD64_DE_CFG`, FXSAVE-leak bug, ARAT for fam>=0x12, Translation-Cache-Extension (`MSR_EFER.TCE`), AutoIBRS, and SYSRET-SS-attrs bug. Per-`bsp_init_amd(c)` runs once on BSP to set up VA-align (fam 0x15 cache-aliasing entropy), Zen generation feature caps, SNP detection (`bsp_determine_snp` + `snp_probe_rmptable_info`), TSA mitigation (`tsa_init`), CPUID-FAULT, and SSBD-via-LS_CFG bit. Per-`early_detect_mem_encrypt(c)` reads `MSR_AMD64_SYSCFG.MEM_ENCRYPT` and `MSR_K7_HWCR.SMMLOCK`, adjusts `x86_phys_bits` by the SME C-bit, and gates SME/SEV/SEV_ES/SEV_SNP caps. Per-`amd_set_dr_addr_mask` / `amd_get_dr_addr_mask` provide BPEXT debug-register-address-mask MSRs `MSR_F16H_DR{0..3}_ADDR_MASK`. Critical for: correct AMD-uarch quirks, dependable encrypted-memory gating, defense against broken-BIOS / broken-microcode states.

This Tier-3 covers `arch/x86/kernel/cpu/amd.c` (~1434 lines).

### Acceptance Criteria

- [ ] AC-1: AMD vendor cpu_dev is link-registered; cpuid "AuthenticAMD" routes to early_init_amd + bsp_init_amd + init_amd + cpu_detect_tlb_amd.
- [ ] AC-2: early_init_amd sets K8 cap when fam>=0xf, latches microcode from MSR_AMD64_PATCH_LEVEL, and sets CONSTANT_TSC/NONSTOP_TSC/ACC_POWER/RAPL bit-for-bit from 8000_0007 EDX.
- [ ] AC-3: F16h model<=0xf: MSR_AMD64_LS_CFG bit 15 set (erratum 793).
- [ ] AC-4: !HYPERVISOR ∧ !IBPB_BRTYPE ∧ (fam==0x17 ∧ AMD_IBPB ∨ fam>=0x19 ∧ wrmsrq_safe(MSR_IA32_PRED_CMD, PRED_CMD_SBPB) == 0): IBPB_BRTYPE forced; SBPB forced on the fam>=0x19 path.
- [ ] AC-5: early_detect_mem_encrypt with MSR_AMD64_SYSCFG.MEM_ENCRYPT cleared OR CONFIG_X86_32: SME ∧ SEV ∧ SEV_ES ∧ SEV_SNP all cleared.
- [ ] AC-6: early_detect_mem_encrypt with MEM_ENCRYPT set: x86_phys_bits -= cbit; if !MSR_K7_HWCR.SMMLOCK ⟹ SEV/SEV_ES/SEV_SNP cleared (SME retained).
- [ ] AC-7: bsp_init_amd: fam==0x15 — va_align.{mask, flags, bits} populated from CPUID 0x80000005 + get_random_u32().
- [ ] AC-8: bsp_init_amd: !AMD_SSBD ∧ !VIRT_SSBD ∧ fam ∈ {0x15..0x17} ⟹ LS_CFG_SSBD + SSBD forced; mask bit set per family.
- [ ] AC-9: bsp_init_amd Zen-generation matrix: fam 0x17/0x19/0x1a model ranges map to ZEN1..ZEN6 exactly; out-of-range emits WARN_ONCE.
- [ ] AC-10: bsp_determine_snp: SEV_SNP set in CPUID ∧ !HYPERVISOR ∧ (ZEN3 ∨ ZEN4 ∨ RMPREAD) ∧ snp_probe_rmptable_info() ⟹ CC_ATTR_HOST_SEV_SNP set; otherwise SEV_SNP cleared.
- [ ] AC-11: tsa_init: ZEN3/ZEN4 with matching microcode ⟹ VERW_CLEAR forced; else (non-ZEN3/4 non-HYPERVISOR) ⟹ TSA_SQ_NO + TSA_L1_NO forced.
- [ ] AC-12: init_amd: per-family dispatch correct: 4 ⟹ k5, 5 ⟹ k6, 6 ⟹ k7, 0xf ⟹ k8, 0x10 ⟹ gh, 0x12 ⟹ ln, 0x15 ⟹ bd, 0x16 ⟹ jg; fam>=0x17 ⟹ init_amd_zen_common + appropriate init_amd_zenN by ZEN-feature cap.
- [ ] AC-13: init_amd: MSR_VM_CR.SVM_DIS set ⟹ SVM cleared; LFENCE-RDTSC enabled via MSR_AMD64_DE_CFG bit when !LFENCE_RDTSC ∧ XMM2.
- [ ] AC-14: init_amd_k8: erratum #110 LAHF_LM cleared when model<0x14 ∧ LAHF_LM ∧ !HYPERVISOR; HWCR bit 6 set for SMP TLB-flush-filter disable; X86_BUG_SWAPGS_FENCE set.
- [ ] AC-15: init_amd_gh: GART TLB Walk Errors masked (MSR_AMD64_MCx_MASK(4) bit 10); BU_CFG2 bit 24 cleared; X86_BUG_AMD_TLB_MMATCH set.
- [ ] AC-16: init_amd_zen1: erratum 1076 ⟹ CPB set; X86_BUG_DIV0 forced; model<0x30 ⟹ IRPERF cleared + MSR_K7_HWCR IRPERF disabled; FP_CFG ZEN1 denorm fix set.
- [ ] AC-17: init_amd_zen2: spectral chicken bit set when !HYPERVISOR; zenbleed mitigation MSR_AMD64_DE_CFG bit set/cleared per microcode match; Cyan Skillfish (0x47/0x0) ⟹ RDSEED cleared + CPUID_FN_7 bit 18 cleared; INVLPGB cleared.
- [ ] AC-18: init_amd_zen3: BTC_NO synthesized when not set in CPUID and !HYPERVISOR.
- [ ] AC-19: init_amd_zen4: shared-BTB-fix bit set !HYPERVISOR; V_VMSAVE_VMLOAD cleared on model 0x18..0x1f and 0x60..0x7f.
- [ ] AC-20: init_amd_zen5: RDSEED cleared + CPUID_FN_7 bit 18 cleared when microcode does not match zen5_rdseed_microcode[].
- [ ] AC-21: cpu_detect_tlb_amd: tlb_lli_4k / tlb_lld_4k from CPUID 0x80000006 EBX; 2M values fall back to CPUID 0x80000005 when L2 disabled; INVLPGB cap ⟹ invlpgb_count_max = (cpuid_edx(0x80000008) & 0xffff) + 1.
- [ ] AC-22: amd_set_dr_addr_mask: BPEXT cap required; dr<4; only writes MSR when cached value differs.
- [ ] AC-23: amd_check_microcode: ZEN2 ⟹ on_each_cpu(zenbleed_check_cpu).
- [ ] AC-24: rdmsrq_amd_safe / wrmsrq_amd_safe: WARN_ONCE if boot_cpu_data.x86 != 0xf; password (gprs[7]) == 0x9c5a203a.
- [ ] AC-25: print_s5_reset_status_mmio: gated on ZEN; ignores U32_MAX; W1C clears the register; logs each set bit per s5_reset_reason_txt.

### Architecture

```
struct AmdCpuDev {
  vendor: &'static str,                // "AMD"
  ident: &'static [&'static str],      // { "AuthenticAMD" }
  x86_vendor: u8,                      // X86_VENDOR_AMD
  early_init: fn(&mut CpuInfoX86),     // Amd::early_init
  bsp_init:   fn(&mut CpuInfoX86),     // Amd::bsp_init
  init:       fn(&mut CpuInfoX86),     // Amd::init
  detect_tlb: fn(&mut CpuInfoX86),     // Amd::detect_tlb
  legacy_models:     Option<LegacyModels>,    // X86_32 only
  legacy_cache_size: Option<fn(&CpuInfoX86, u32) -> u32>, // X86_32 only
}

static AMD_TSA_MICROCODE: &[X86CpuIdStep]    = &[ /* fam 0x19 model/stepping/ucode tuples */ ];
static AMD_ZENBLEED_MICROCODE: &[X86CpuIdStep] = &[ /* fam 0x17 model/stepping/ucode tuples */ ];
static ERRATUM_1386_MICROCODE: &[X86CpuIdStep] = &[ /* fam 0x17 model/stepping/ucode tuples */ ];
static ZEN5_RDSEED_MICROCODE: &[X86CpuIdStep]  = &[ /* fam 0x1a model/stepping/ucode tuples */ ];

static AMD_MSR_DR_ADDR_MASKS: [u32; 4] = [
  MSR_F16H_DR0_ADDR_MASK,
  MSR_F16H_DR1_ADDR_MASK,
  MSR_F16H_DR1_ADDR_MASK + 1,
  MSR_F16H_DR1_ADDR_MASK + 2,
];

static S5_RESET_REASON_TXT: &[Option<&str>] = &[ /* bit-indexed cause strings, 0..31 */ ];
```

`Amd::early_init(c)`:
1. if c.x86 >= 0xf: set X86_FEATURE_K8.
2. rdmsr_safe(MSR_AMD64_PATCH_LEVEL, &c.microcode, &dummy).
3. if c.x86_power & (1 << 8): set CONSTANT_TSC ∧ NONSTOP_TSC.
4. if c.x86_power & BIT(12): set ACC_POWER.
5. if c.x86_power & BIT(14): set RAPL.
6. if X86_64: set SYSCALL32.
7. else if c.x86 == 5 ∧ (model ∈ {13, 9} ∨ (model == 8 ∧ stepping >= 8)): set K6_MTRR.
8. if boot_cpu_has(APIC): EXTD_APICID derivation (fam>0x16 or PCI-config 0:24:0/0x68 bits 17:18 == 3).
9. set X86_FEATURE_VMMCALL.
10. if c.x86 == 0x16 ∧ c.x86_model <= 0xf: msr_set_bit(MSR_AMD64_LS_CFG, 15) /* erratum 793 */.
11. Amd::early_detect_mem_encrypt(c).
12. if !HYPERVISOR ∧ !IBPB_BRTYPE: IBPB_BRTYPE / SBPB synthesis (fam 0x17 AMD_IBPB or fam>=0x19 PRED_CMD_SBPB).

`Amd::bsp_init(c)`:
1. if CONSTANT_TSC ∧ (c.x86 > 0x10 ∨ (c.x86 == 0x10 ∧ c.x86_model >= 0x2)): rdmsrq(MSR_K7_HWCR, val); if !(val & BIT(24)): pr_warn(FW_BUG ...).
2. if c.x86 == 0x15: VA-align entropy from CPUID 0x80000005.
3. if cpu_has(c, MWAITX): use_mwaitx_delay().
4. if !AMD_SSBD ∧ !VIRT_SSBD ∧ 0x15 <= c.x86 <= 0x17: LS_CFG SSBD bit per fam (54/33/10); force LS_CFG_SSBD + SSBD.
5. resctrl_cpu_detect(c).
6. Zen-generation matrix (0x17/0x19/0x1a model-range → ZEN1..ZEN6).
7. Amd::bsp_determine_snp(c).
8. Amd::tsa_init(c).
9. if cpu_has(c, GP_ON_USER_CPUID): setup_force_cpu_cap(CPUID_FAULT).

`Amd::init(c)`:
1. Amd::early_init(c).
2. if c.x86 >= 0x10: set REP_GOOD.
3. if cpu_has(c, FSRM): set FSRS.
4. if c.x86 < 6: clear MCE.
5. match c.x86: 4 ⟹ k5; 5 ⟹ k6; 6 ⟹ k7; 0xf ⟹ k8; 0x10 ⟹ gh; 0x12 ⟹ ln; 0x15 ⟹ bd; 0x16 ⟹ jg.
6. if c.x86 >= 0x17: Amd::init_zen_common().
7. dispatch ZEN1..ZEN5 per boot_cpu_has.
8. if c.x86 >= 6 ∧ !cpu_has(c, XSAVEERPTR): set X86_BUG_FXSAVE_LEAK.
9. cpu_detect_cache_sizes(c); Amd::srat_detect_node(c); init_amd_cacheinfo(c).
10. if cpu_has(c, SVM) ∧ (MSR_VM_CR & SVM_VM_CR_SVM_DIS_MASK): clear SVM.
11. if !LFENCE_RDTSC ∧ XMM2: msr_set_bit(MSR_AMD64_DE_CFG, LFENCE_SERIALIZE_BIT); set LFENCE_RDTSC.
12. if c.x86 > 0x11: set ARAT.
13. if !3DNOWPREFETCH ∧ (3DNOW ∨ LM): set 3DNOWPREFETCH.
14. if !XENPV: set X86_BUG_SYSRET_SS_ATTRS.
15. if IRPERF: msr_set_bit(MSR_K7_HWCR, IRPERF_EN_BIT).
16. check_null_seg_clears_base(c).
17. if spectre_v2_in_eibrs_mode(...) ∧ AUTOIBRS: msr_set_bit(MSR_EFER, _EFER_AUTOIBRS).
18. clear APIC_MSRS_FENCE.
19. if cpu_has(c, TCE): msr_set_bit(MSR_EFER, _EFER_TCE).

`Amd::early_detect_mem_encrypt(c)`:
1. if c.extended_cpuid_level >= 0x8000001f ∧ (cpuid_eax(0x8000001f) & BIT(0)):
   - __this_cpu_write(cache_state_incoherent, true).
2. if SME ∨ SEV:
   - rdmsrq(MSR_AMD64_SYSCFG, msr).
   - if !(msr & MSR_AMD64_SYSCFG_MEM_ENCRYPT): goto clear_all.
   - c.x86_phys_bits -= (cpuid_ebx(0x8000001f) >> 6) & 0x3f.
   - if X86_32: goto clear_all.
   - if !sme_me_mask: setup_clear_cpu_cap(SME).
   - rdmsrq(MSR_K7_HWCR, msr); if !(msr & MSR_K7_HWCR_SMMLOCK): goto clear_sev.
   - return.
3. clear_all: setup_clear_cpu_cap(SME).
4. clear_sev: setup_clear_cpu_cap(SEV), setup_clear_cpu_cap(SEV_ES), setup_clear_cpu_cap(SEV_SNP).

`Amd::bsp_determine_snp(c)`:
1. cc_vendor = CC_VENDOR_AMD.
2. if cpu_has(c, SEV_SNP):
   - if !HYPERVISOR ∧ (ZEN3 ∨ ZEN4 ∨ RMPREAD) ∧ snp_probe_rmptable_info(): cc_platform_set(CC_ATTR_HOST_SEV_SNP).
   - else: setup_clear_cpu_cap(SEV_SNP); cc_platform_clear(CC_ATTR_HOST_SEV_SNP).

`Amd::tsa_init(c)`:
1. if HYPERVISOR: return.
2. if ZEN3 ∨ ZEN4:
   - if x86_match_min_microcode_rev(amd_tsa_microcode): setup_force_cpu_cap(VERW_CLEAR).
   - else: pr_debug.
3. else: setup_force_cpu_cap(TSA_SQ_NO); setup_force_cpu_cap(TSA_L1_NO).

`Amd::init_k8(c)` (selected):
1. level = cpuid_eax(1); if (0x0f48..0x0f50) ∨ level >= 0x0f58: set REP_GOOD.
2. if c.x86_model < 0x14 ∧ LAHF_LM ∧ !HYPERVISOR:
   - clear LAHF_LM.
   - if rdmsrq_amd_safe(0xc001100d, &value) == 0: value &= ~BIT_64(32); wrmsrq_amd_safe(0xc001100d, value).
3. if c.x86_model_id is empty: strscpy(c.x86_model_id, "Hammer").
4. if SMP: msr_set_bit(MSR_K7_HWCR, 6) /* TLB flush filter disable */.
5. set X86_BUG_SWAPGS_FENCE.
6. if c.x86_model > 0x41 ∨ (c.x86_model == 0x41 ∧ c.x86_stepping >= 0x2): setup_force_cpu_bug(X86_BUG_AMD_E400).

`Amd::init_zen1(c)`:
1. Amd::fix_erratum_1386(c).
2. if !HYPERVISOR ∧ !CPB: set CPB.
3. setup_force_cpu_bug(X86_BUG_DIV0).
4. if c.x86_model < 0x30: msr_clear_bit(MSR_K7_HWCR, IRPERF_EN_BIT); clear IRPERF.
5. msr_set_bit(MSR_AMD64_FP_CFG, ZEN1_DENORM_FIX_BIT).

`Amd::init_zen2(c)`:
1. Amd::init_spectral_chicken(c).
2. Amd::fix_erratum_1386(c).
3. Amd::zen2_zenbleed_check(c).
4. if c.x86_model == 0x47 ∧ c.x86_stepping == 0: clear RDSEED; msr_clear_bit(MSR_AMD64_CPUID_FN_7, 18).
5. clear INVLPGB.

`Amd::init_zen3(c)`:
1. if !HYPERVISOR ∧ !BTC_NO: set BTC_NO.

`Amd::init_zen4(c)`:
1. if !HYPERVISOR: msr_set_bit(MSR_ZEN4_BP_CFG, SHARED_BTB_FIX_BIT).
2. match c.x86_model in {0x18..0x1f, 0x60..0x7f}: clear V_VMSAVE_VMLOAD.

`Amd::init_zen5(c)`:
1. if !x86_match_min_microcode_rev(zen5_rdseed_microcode): clear RDSEED; msr_clear_bit(MSR_AMD64_CPUID_FN_7, 18).

`Amd::set_dr_addr_mask(mask, dr)`:
1. if !BPEXT: return.
2. WARN_ON_ONCE(dr >= 4).
3. if per_cpu(amd_dr_addr_mask, cpu)[dr] == mask: return.
4. wrmsrq(amd_msr_dr_addr_masks[dr], mask).
5. per_cpu(amd_dr_addr_mask, cpu)[dr] = mask.

`Amd::detect_tlb(c)`:
1. if c.x86 < 0xf: return.
2. if c.extended_cpuid_level < 0x80000006: return.
3. cpuid(0x80000006, &eax, &ebx, &ecx, &edx); mask = 0xfff.
4. tlb_lld_4k = (ebx >> 16) & mask; tlb_lli_4k = ebx & mask.
5. if c.x86 == 0xf: cpuid(0x80000005, ...); mask = 0xff.
6. DTLB/ITLB 2M derivation as per REQ-24.
7. tlb_lld_4m = tlb_lld_2m >> 1; tlb_lli_4m = tlb_lli_2m >> 1.
8. if INVLPGB: invlpgb_count_max = (cpuid_edx(0x80000008) & 0xffff) + 1.

### Out of Scope

- arch/x86/kernel/cpu/common.c (covered in `cpu-common.md` Tier-3)
- arch/x86/kernel/cpu/bugs.c / Spectre/Retbleed/SRSO mitigations (covered in `cpu-mitigations.md` Tier-3)
- arch/x86/kernel/cpu/cacheinfo.c / init_amd_cacheinfo (covered separately if expanded)
- arch/x86/kernel/cpu/resctrl/* (covered separately if expanded)
- arch/x86/mm/mem_encrypt*.c SME/SEV runtime (covered separately if expanded)
- arch/x86/virt/svm/sev.c SEV host driver / snp_probe_rmptable_info implementation (covered separately if expanded)
- arch/x86/kernel/amd_nb.c northbridge enumeration (covered separately if expanded)
- arch/x86/events/amd/* PMU drivers (covered separately if expanded)
- drivers/cpufreq/amd-pstate.c (covered separately if expanded — this Tier-3 only notes the feature caps it depends on, ACC_POWER / RAPL / CPB / ARAT)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cpu_dev amd_cpu_dev` | per-vendor dispatch | `AmdCpuDev` |
| `cpu_dev_register(amd_cpu_dev)` | per-link-time registration | `AmdCpuDev::register` |
| `early_init_amd(c)` | per-CPU pre-enum init | `Amd::early_init` |
| `init_amd(c)` | per-CPU post-enum init | `Amd::init` |
| `bsp_init_amd(c)` | per-BSP-only init | `Amd::bsp_init` |
| `cpu_detect_tlb_amd(c)` | per-CPU TLB enumeration | `Amd::detect_tlb` |
| `amd_size_cache(c, size)` | per-CPU legacy cache-size override | `Amd::size_cache` |
| `init_amd_k5(c)` | per-family-4 (Elan) init | `Amd::init_k5` |
| `init_amd_k6(c)` | per-family-5 (K6 WHCR / B-step) init | `Amd::init_k6` |
| `init_amd_k7(c)` | per-family-6 (Athlon CLK_CTL / SMP cert) init | `Amd::init_k7` |
| `init_amd_k8(c)` | per-family-0xf (K8) init | `Amd::init_k8` |
| `init_amd_gh(c)` | per-family-0x10 (Greyhound / Fam10h) init | `Amd::init_gh` |
| `init_amd_ln(c)` | per-family-0x12 (Llano) init | `Amd::init_ln` |
| `init_amd_bd(c)` | per-family-0x15 (Bulldozer) init | `Amd::init_bd` |
| `init_amd_jg(c)` | per-family-0x16 (Jaguar) init | `Amd::init_jg` |
| `init_amd_zen_common()` | per-Zen common init | `Amd::init_zen_common` |
| `init_amd_zen1(c)` | per-Zen1 init (erratum 1076, DIV0, IRPERF) | `Amd::init_zen1` |
| `init_amd_zen2(c)` | per-Zen2 init (Spectral-Chicken, Zenbleed, erratum 1386) | `Amd::init_zen2` |
| `init_amd_zen3(c)` | per-Zen3 init (BTC_NO synthesis) | `Amd::init_zen3` |
| `init_amd_zen4(c)` | per-Zen4 init (SHARED_BTB_FIX, V_VMSAVE_VMLOAD) | `Amd::init_zen4` |
| `init_amd_zen5(c)` | per-Zen5 init (RDSEED32 broken) | `Amd::init_zen5` |
| `init_spectral_chicken(c)` | per-Zen2 chicken bit | `Amd::init_spectral_chicken` |
| `zen2_zenbleed_check(c)` | per-Zen2 zenbleed fixup | `Amd::zen2_zenbleed_check` |
| `fix_erratum_1386(c)` | per-Zen1/2 XSAVES erratum | `Amd::fix_erratum_1386` |
| `clear_rdrand_cpuid_bit(c)` | per-CPU RDRAND suspend/resume gate | `Amd::clear_rdrand_cpuid_bit` |
| `bsp_determine_snp(c)` | per-BSP SNP detect | `Amd::bsp_determine_snp` |
| `tsa_init(c)` | per-BSP TSA mitigation | `Amd::tsa_init` |
| `early_detect_mem_encrypt(c)` | per-CPU SME/SEV gate | `Amd::early_detect_mem_encrypt` |
| `srat_detect_node(c)` | per-CPU NUMA-node fixup | `Amd::srat_detect_node` |
| `nearby_node(apicid)` | per-broken-NUMA fallback | `Amd::nearby_node` |
| `rdmsrq_amd_safe(msr, *p)` | per-K8 MSR-safe rdmsr | `Amd::rdmsrq_amd_safe` |
| `wrmsrq_amd_safe(msr, val)` | per-K8 MSR-safe wrmsr | `Amd::wrmsrq_amd_safe` |
| `amd_set_dr_addr_mask(mask, dr)` | per-CPU BPEXT DR-mask write | `Amd::set_dr_addr_mask` |
| `amd_get_dr_addr_mask(dr)` | per-CPU BPEXT DR-mask read | `Amd::get_dr_addr_mask` |
| `amd_check_microcode()` | per-system ucode revalidation | `Amd::check_microcode` |
| `zenbleed_check_cpu(unused)` | per-CPU IPI body | `Amd::zenbleed_check_cpu` |
| `print_s5_reset_status_mmio()` | per-Zen S5 reset reason | `Amd::print_s5_reset_status_mmio` |
| `print_dmi_agesa()` | per-AGESA version log | `Amd::print_dmi_agesa` |
| `amd_tsa_microcode[]` | per-table TSA ucode | `Amd::AMD_TSA_MICROCODE` |
| `amd_zenbleed_microcode[]` | per-table zenbleed ucode | `Amd::AMD_ZENBLEED_MICROCODE` |
| `erratum_1386_microcode[]` | per-table erratum-1386 ucode | `Amd::ERRATUM_1386_MICROCODE` |
| `zen5_rdseed_microcode[]` | per-table Zen5 RDSEED ucode | `Amd::ZEN5_RDSEED_MICROCODE` |
| `rdrand_force` | per-cmdline RDRAND force | `Amd::rdrand_force` |
| `invlpgb_count_max` | per-system INVLPGB pages | `Amd::invlpgb_count_max` |
| `amd_dr_addr_mask` | per-CPU DR-mask cache | `Amd::dr_addr_mask` |
| `amd_msr_dr_addr_masks[]` | per-DR MSR index table | `Amd::AMD_MSR_DR_ADDR_MASKS` |
| `s5_reset_reason_txt[]` | per-bit S5 reason string | `Amd::S5_RESET_REASON_TXT` |

### compatibility contract

REQ-1: struct amd_cpu_dev:
- c_vendor = "AMD".
- c_ident = { "AuthenticAMD" }.
- c_x86_vendor = X86_VENDOR_AMD.
- c_early_init = early_init_amd.
- c_bsp_init = bsp_init_amd.
- c_init = init_amd.
- c_detect_tlb = cpu_detect_tlb_amd.
- legacy_models / legacy_cache_size: per-CONFIG_X86_32 only.

REQ-2: early_init_amd(c):
- /* K8 family marker */
- if c.x86 >= 0xf: set X86_FEATURE_K8.
- /* Microcode revision */
- rdmsr_safe(MSR_AMD64_PATCH_LEVEL, &c.microcode, &dummy).
- /* Constant / non-stop TSC, ACC_POWER, RAPL from 8000_0007 EDX */
- if c.x86_power & (1 << 8): set CONSTANT_TSC ∧ NONSTOP_TSC.
- if c.x86_power & BIT(12): set ACC_POWER.
- if c.x86_power & BIT(14): set RAPL.
- /* X86_64: SYSCALL32 unconditional */
- if CONFIG_X86_64: set SYSCALL32.
- else if c.x86 == 5 ∧ (model ∈ {13, 9} ∨ (model == 8 ∧ stepping >= 8)): set K6_MTRR.
- /* Extended APIC ID */
- if boot_cpu_has(APIC):
  - if c.x86 > 0x16: set EXTD_APICID.
  - else if c.x86 >= 0xf:
    - val = read_pci_config(0, 24, 0, 0x68).
    - if ((val >> 17) & 0x3) == 0x3: set EXTD_APICID.
- /* VMMCALL is set unconditionally — only relevant under virt */
- set X86_FEATURE_VMMCALL.
- /* F16h erratum 793 (CVE-2013-6885) */
- if c.x86 == 0x16 ∧ c.x86_model <= 0xf: msr_set_bit(MSR_AMD64_LS_CFG, 15).
- early_detect_mem_encrypt(c).
- /* IBPB_BRTYPE / SBPB synthesis */
- if !HYPERVISOR ∧ !IBPB_BRTYPE:
  - if c.x86 == 0x17 ∧ boot_cpu_has(AMD_IBPB): setup_force_cpu_cap(IBPB_BRTYPE).
  - else if c.x86 >= 0x19 ∧ !wrmsrq_safe(MSR_IA32_PRED_CMD, PRED_CMD_SBPB):
    - setup_force_cpu_cap(IBPB_BRTYPE).
    - setup_force_cpu_cap(SBPB).

REQ-3: bsp_init_amd(c):
- /* TSC P0-frequency sanity (fam > 0x10 ∨ fam==0x10 ∧ model >= 0x2) */
- if cpu_has(c, CONSTANT_TSC):
  - if c.x86 > 0x10 ∨ (c.x86 == 0x10 ∧ c.x86_model >= 0x2):
    - rdmsrq(MSR_K7_HWCR, val).
    - if !(val & BIT(24)): pr_warn(FW_BUG "TSC doesn't count with P0 frequency!").
- /* Fam 0x15 VA-align cache-aliasing entropy */
- if c.x86 == 0x15:
  - cpuid = cpuid_edx(0x80000005).
  - assoc = (cpuid >> 16) & 0xff.
  - upperbit = ((cpuid >> 24) << 10) / assoc.
  - va_align.mask = (upperbit - 1) & PAGE_MASK.
  - va_align.flags = ALIGN_VA_32 | ALIGN_VA_64.
  - va_align.bits = get_random_u32() & va_align.mask.
- if cpu_has(c, MWAITX): use_mwaitx_delay().
- /* SSBD via LS_CFG bit (fam 0x15..0x17) */
- if !AMD_SSBD ∧ !VIRT_SSBD ∧ 0x15 <= c.x86 <= 0x17:
  - switch c.x86: 0x15 ⟹ bit 54; 0x16 ⟹ bit 33; 0x17 ⟹ bit 10.
  - if rdmsrq_safe(MSR_AMD64_LS_CFG, &x86_amd_ls_cfg_base) == 0:
    - setup_force_cpu_cap(LS_CFG_SSBD).
    - setup_force_cpu_cap(SSBD).
    - x86_amd_ls_cfg_ssbd_mask = 1ULL << bit.
- resctrl_cpu_detect(c).
- /* Zen-generation feature force */
- switch c.x86:
  - 0x17: model ∈ {0x00..0x2f, 0x50..0x5f} ⟹ ZEN1; {0x30..0x4f, 0x60..0x7f, 0x90..0x91, 0xa0..0xaf} ⟹ ZEN2; else WARN.
  - 0x19: model ∈ {0x00..0x0f, 0x20..0x5f} ⟹ ZEN3; {0x10..0x1f, 0x60..0xaf} ⟹ ZEN4; else WARN.
  - 0x1a: model ∈ {0x00..0x2f, 0x40..0x4f, 0x60..0x7f} ⟹ ZEN5; {0x50..0x5f, 0x80..0xaf, 0xc0..0xcf} ⟹ ZEN6; else WARN.
  - default: skip.
- bsp_determine_snp(c).
- tsa_init(c).
- if cpu_has(c, GP_ON_USER_CPUID): setup_force_cpu_cap(CPUID_FAULT).

REQ-4: early_detect_mem_encrypt(c):
- /* SME-supported processors require WBINVD on kexec */
- if c.extended_cpuid_level >= 0x8000001f ∧ (cpuid_eax(0x8000001f) & BIT(0)):
  - __this_cpu_write(cache_state_incoherent, true).
- /* BIOS-enabled SME/SEV */
- if cpu_has(c, SME) ∨ cpu_has(c, SEV):
  - rdmsrq(MSR_AMD64_SYSCFG, msr).
  - if !(msr & MSR_AMD64_SYSCFG_MEM_ENCRYPT): goto clear_all.
  - c.x86_phys_bits -= (cpuid_ebx(0x8000001f) >> 6) & 0x3f.
  - if CONFIG_X86_32: goto clear_all.
  - if !sme_me_mask: setup_clear_cpu_cap(SME).
  - rdmsrq(MSR_K7_HWCR, msr).
  - if !(msr & MSR_K7_HWCR_SMMLOCK): goto clear_sev.
  - return.
- clear_all: setup_clear_cpu_cap(SME).
- clear_sev: setup_clear_cpu_cap(SEV), setup_clear_cpu_cap(SEV_ES), setup_clear_cpu_cap(SEV_SNP).

REQ-5: bsp_determine_snp(c):
- /* per-CONFIG_ARCH_HAS_CC_PLATFORM only */
- cc_vendor = CC_VENDOR_AMD.
- if cpu_has(c, SEV_SNP):
  - if !HYPERVISOR ∧ (ZEN3 ∨ ZEN4 ∨ RMPREAD) ∧ snp_probe_rmptable_info():
    - cc_platform_set(CC_ATTR_HOST_SEV_SNP).
  - else:
    - setup_clear_cpu_cap(SEV_SNP); cc_platform_clear(CC_ATTR_HOST_SEV_SNP).

REQ-6: tsa_init(c):
- if cpu_has(c, HYPERVISOR): return.
- if cpu_has(c, ZEN3) ∨ cpu_has(c, ZEN4):
  - if x86_match_min_microcode_rev(amd_tsa_microcode): setup_force_cpu_cap(VERW_CLEAR).
  - else: pr_debug.
- else:
  - setup_force_cpu_cap(TSA_SQ_NO); setup_force_cpu_cap(TSA_L1_NO).

REQ-7: init_amd(c):
- early_init_amd(c).
- if c.x86 >= 0x10: set REP_GOOD.
- if cpu_has(c, FSRM): set FSRS.
- /* K6s falsely report MCE */
- if c.x86 < 6: clear MCE.
- /* Family dispatch */
- switch c.x86: 4 ⟹ k5; 5 ⟹ k6; 6 ⟹ k7; 0xf ⟹ k8; 0x10 ⟹ gh; 0x12 ⟹ ln; 0x15 ⟹ bd; 0x16 ⟹ jg.
- if c.x86 >= 0x17: init_amd_zen_common().
- if boot_cpu_has(ZEN1): init_amd_zen1(c).
- else if boot_cpu_has(ZEN2): init_amd_zen2(c).
- else if boot_cpu_has(ZEN3): init_amd_zen3(c).
- else if boot_cpu_has(ZEN4): init_amd_zen4(c).
- else if boot_cpu_has(ZEN5): init_amd_zen5(c).
- /* FXSAVE leak: fam>=6 ∧ !XSAVEERPTR */
- if c.x86 >= 6 ∧ !cpu_has(c, XSAVEERPTR): set X86_BUG_FXSAVE_LEAK.
- cpu_detect_cache_sizes(c).
- srat_detect_node(c).
- init_amd_cacheinfo(c).
- /* SVM-disabled-in-BIOS */
- if cpu_has(c, SVM):
  - rdmsrq(MSR_VM_CR, vm_cr).
  - if vm_cr & SVM_VM_CR_SVM_DIS_MASK: clear SVM.
- /* LFENCE-serializing via DE_CFG */
- if !cpu_has(c, LFENCE_RDTSC) ∧ cpu_has(c, XMM2):
  - msr_set_bit(MSR_AMD64_DE_CFG, MSR_AMD64_DE_CFG_LFENCE_SERIALIZE_BIT).
  - set LFENCE_RDTSC.
- /* ARAT — fam>0x11 */
- if c.x86 > 0x11: set ARAT.
- /* 3DNow / LM implies 3DNOWPREFETCH */
- if !cpu_has(c, 3DNOWPREFETCH):
  - if cpu_has(c, 3DNOW) ∨ cpu_has(c, LM): set 3DNOWPREFETCH.
- /* AMD SYSRET-SS-attrs bug (Xen exempt) */
- if !cpu_feature_enabled(XENPV): set X86_BUG_SYSRET_SS_ATTRS.
- if cpu_has(c, IRPERF): msr_set_bit(MSR_K7_HWCR, MSR_K7_HWCR_IRPERF_EN_BIT).
- check_null_seg_clears_base(c).
- /* AutoIBRS guarantee */
- if spectre_v2_in_eibrs_mode(spectre_v2_enabled) ∧ cpu_has(c, AUTOIBRS):
  - WARN_ON_ONCE(msr_set_bit(MSR_EFER, _EFER_AUTOIBRS) < 0).
- clear APIC_MSRS_FENCE.
- if cpu_has(c, TCE): msr_set_bit(MSR_EFER, _EFER_TCE).

REQ-8: init_amd_k5/k6/k7 (X86_32):
- init_amd_k5: Elan model 9/10 — disable CBAR alias on (CBAR & CBAR_ENB) ⟹ outl(0|CBAR_KEY, CBAR).
- init_amd_k6: K6 B-step BIST loop (model 6, stepping 1) via vide(); WHCR write-allocate (old style mbytes<=508; new style mbytes<=4092).
- init_amd_k7: MSR_K7_HWCR bit 15 clear ⟹ enable SSE on Palomino/Morgan/Barton (models 6..10); MSR_K7_CLK_CTL = 0x200xxxxx for model>=8 stepping>=1 or model>8; AMD MP-cert check else WARN_ONCE + TAINT_CPU_OUT_OF_SPEC.

REQ-9: init_amd_k8:
- /* C+ stepping rep ucode */
- level = cpuid_eax(1).
- if (level >= 0x0f48 ∧ level < 0x0f50) ∨ level >= 0x0f58: set REP_GOOD.
- /* Erratum #110 — LAHF/SAHF only on K8 rev-D and later */
- if c.x86_model < 0x14 ∧ cpu_has(c, LAHF_LM) ∧ !HYPERVISOR:
  - clear LAHF_LM.
  - if rdmsrq_amd_safe(0xc001100d, &value) == 0:
    - value &= ~BIT_64(32); wrmsrq_amd_safe(0xc001100d, value).
- if !c.x86_model_id[0]: strscpy(c.x86_model_id, "Hammer").
- /* TLB flush filter disable (Errata 63 / 122) */
- if CONFIG_SMP: msr_set_bit(MSR_K7_HWCR, 6).
- set X86_BUG_SWAPGS_FENCE.
- /* Erratum 400 (idle) */
- if c.x86_model > 0x41 ∨ (c.x86_model == 0x41 ∧ c.x86_stepping >= 0x2): setup_force_cpu_bug(X86_BUG_AMD_E400).

REQ-10: init_amd_gh (Fam10h):
- /* MMCONF DMI check on BSP */
- if c == &boot_cpu_data ∧ CONFIG_MMCONF_FAM10H: check_enable_amd_mmconf_dmi(); fam10h_check_enable_mmcfg().
- /* GART TLB Walk Errors mask: MSR_AMD64_MCx_MASK(4) bit 10 */
- msr_set_bit(MSR_AMD64_MCx_MASK(4), 10).
- /* WC+ corruption — MSR_AMD64_BU_CFG2 bit 24 clear */
- msr_clear_bit(MSR_AMD64_BU_CFG2, 24).
- set X86_BUG_AMD_TLB_MMATCH.
- if c.x86_model > 0x2 ∨ (c.x86_model == 0x2 ∧ c.x86_stepping >= 0x1): setup_force_cpu_bug(X86_BUG_AMD_E400).

REQ-11: init_amd_ln (Llano F12):
- msr_set_bit(MSR_AMD64_DE_CFG, 31) /* erratum 665 */.

REQ-12: init_amd_bd (Bulldozer F15):
- /* Disable way-access filter on models 0x02..0x1f */
- if 0x02 <= c.x86_model < 0x20:
  - rdmsrq_safe(MSR_F15H_IC_CFG, &value) ∧ !(value & 0x1E) ⟹ value |= 0x1E; wrmsrq_safe(MSR_F15H_IC_CFG, value).
- clear_rdrand_cpuid_bit(c).

REQ-13: init_amd_jg (Jaguar F16):
- clear_rdrand_cpuid_bit(c).

REQ-14: clear_rdrand_cpuid_bit(c):
- if !CONFIG_PM_SLEEP: return.
- if !(cpuid_ecx(1) & BIT(30)) ∨ rdrand_force: return.
- msr_clear_bit(MSR_AMD64_CPUID_FN_1, 62).
- /* Verify hypervisor honored the bit hide */
- if cpuid_ecx(1) & BIT(30): pr_info_once(...); return.
- clear X86_FEATURE_RDRAND.
- pr_info_once(... hidden via CPUID. Use rdrand=force to reenable).

REQ-15: init_amd_zen_common:
- setup_force_cpu_cap(X86_FEATURE_ZEN).
- if CONFIG_NUMA: node_reclaim_distance = 32.

REQ-16: init_amd_zen1(c):
- fix_erratum_1386(c).
- if !HYPERVISOR:
  - if !cpu_has(c, CPB): set CPB /* erratum 1076 */.
- pr_notice_once("AMD Zen1 DIV0 bug detected. Disable SMT for full protection.").
- setup_force_cpu_bug(X86_BUG_DIV0).
- if c.x86_model < 0x30:
  - msr_clear_bit(MSR_K7_HWCR, MSR_K7_HWCR_IRPERF_EN_BIT) /* erratum 1054 */.
  - clear IRPERF.
- pr_notice_once("AMD Zen1 FPDSS bug detected, enabling mitigation.").
- msr_set_bit(MSR_AMD64_FP_CFG, MSR_AMD64_FP_CFG_ZEN1_DENORM_FIX_BIT).

REQ-17: init_amd_zen2(c):
- init_spectral_chicken(c) /* MSR_ZEN2_SPECTRAL_CHICKEN, MSR_ZEN2_SPECTRAL_CHICKEN_BIT — !HYPERVISOR only */.
- fix_erratum_1386(c).
- zen2_zenbleed_check(c).
- /* RDSEED Cyan Skillfish (model 0x47, stepping 0) */
- if c.x86_model == 0x47 ∧ c.x86_stepping == 0x0:
  - clear RDSEED; msr_clear_bit(MSR_AMD64_CPUID_FN_7, 18); pr_emerg("RDSEED is not reliable on this platform; disabling.").
- clear INVLPGB /* misconfigured CPUID */.

REQ-18: init_amd_zen3(c):
- if !HYPERVISOR ∧ !cpu_has(c, BTC_NO): set BTC_NO /* Zen3 not susceptible to BTC, predates BTC_NO allocation */.

REQ-19: init_amd_zen4(c):
- if !HYPERVISOR: msr_set_bit(MSR_ZEN4_BP_CFG, MSR_ZEN4_BP_CFG_SHARED_BTB_FIX_BIT).
- /* V_VMSAVE_VMLOAD may cause host reboots — model 0x18..0x1f, 0x60..0x7f */
- match c.x86_model in {0x18..0x1f, 0x60..0x7f}: clear V_VMSAVE_VMLOAD.

REQ-20: init_amd_zen5(c):
- if !x86_match_min_microcode_rev(zen5_rdseed_microcode):
  - clear RDSEED; msr_clear_bit(MSR_AMD64_CPUID_FN_7, 18); pr_emerg_once("RDSEED32 is broken. Disabling the corresponding CPUID bit.").

REQ-21: fix_erratum_1386(c):
- /* XSAVES malfunction on certain Zen1/2 parts */
- if x86_match_min_microcode_rev(erratum_1386_microcode): return.
- clear X86_FEATURE_XSAVES /* fall back to XSAVEC, equivalent because no supervisor states */.

REQ-22: zen2_zenbleed_check(c):
- if HYPERVISOR: return.
- if !cpu_has(c, AVX): return.
- if !x86_match_min_microcode_rev(amd_zenbleed_microcode):
  - pr_notice_once("Zenbleed: please update your microcode for the most optimal fix").
  - msr_set_bit(MSR_AMD64_DE_CFG, MSR_AMD64_DE_CFG_ZEN2_FP_BACKUP_FIX_BIT).
- else:
  - msr_clear_bit(MSR_AMD64_DE_CFG, MSR_AMD64_DE_CFG_ZEN2_FP_BACKUP_FIX_BIT).

REQ-23: srat_detect_node(c):
- /* per-CONFIG_NUMA */
- cpu = smp_processor_id(); apicid = c.topo.apicid.
- node = numa_cpu_node(cpu).
- if node == NUMA_NO_NODE: node = per_cpu_llc_id(cpu).
- if x86_cpuinit.fixup_cpu_id: x86_cpuinit.fixup_cpu_id(c, node).
- if !node_online(node):
  - ht_nodeid = c.topo.initial_apicid.
  - if __apicid_to_node[ht_nodeid] != NUMA_NO_NODE: node = __apicid_to_node[ht_nodeid].
  - if !node_online(node): node = nearby_node(apicid).
- numa_set_node(cpu, node).

REQ-24: cpu_detect_tlb_amd(c):
- if c.x86 < 0xf: return.
- if c.extended_cpuid_level < 0x80000006: return.
- cpuid(0x80000006, &eax, &ebx, &ecx, &edx); mask = 0xfff.
- tlb_lld_4k = (ebx >> 16) & mask; tlb_lli_4k = ebx & mask.
- /* K8 reads L1 TLB from 0x80000005 */
- if c.x86 == 0xf: cpuid(0x80000005, ...); mask = 0xff.
- /* DTLB 2M/4M fallback to L1 if disabled */
- if !((eax >> 16) & mask): tlb_lld_2m = (cpuid_eax(0x80000005) >> 16) & 0xff.
- else: tlb_lld_2m = (eax >> 16) & mask.
- tlb_lld_4m = tlb_lld_2m >> 1 /* 4M uses 2 × 2M */.
- /* ITLB 2M/4M */
- if !(eax & mask):
  - if c.x86 == 0x15 ∧ c.x86_model <= 0x1f: tlb_lli_2m = 1024 /* erratum 658 */.
  - else: cpuid(0x80000005, ...); tlb_lli_2m = eax & 0xff.
- else: tlb_lli_2m = eax & mask.
- tlb_lli_4m = tlb_lli_2m >> 1.
- /* INVLPGB max page count */
- if cpu_has(c, INVLPGB): invlpgb_count_max = (cpuid_edx(0x80000008) & 0xffff) + 1.

REQ-25: amd_set_dr_addr_mask(mask, dr) / amd_get_dr_addr_mask(dr):
- /* BPEXT guard */
- if !cpu_feature_enabled(BPEXT): return / return 0.
- WARN_ON_ONCE(dr >= 4).
- amd_msr_dr_addr_masks[dr] = MSR_F16H_DR{0..3}_ADDR_MASK.
- set: skip if cached value matches; wrmsrq + cache.
- get: per_cpu(amd_dr_addr_mask[dr], smp_processor_id()).

REQ-26: amd_check_microcode():
- if boot_cpu_data.x86_vendor != X86_VENDOR_AMD: return.
- if cpu_feature_enabled(ZEN2): on_each_cpu(zenbleed_check_cpu, NULL, 1).

REQ-27: amd_size_cache(c, size) [X86_32]:
- /* AMD errata T13 */
- if c.x86 == 6:
  - Duron Rev A0 (model 3 stepping 0): size = 64.
  - Tbird A1/A2 (model 4 stepping 0/1): size = 256.
- return size.

REQ-28: print_s5_reset_status_mmio (late_initcall):
- if !ZEN: return 0.
- ioremap(FCH_PM_BASE + FCH_PM_S5_RESET_STATUS).
- value = ioread32.
- if value == U32_MAX: iounmap, return 0 /* invalid response */.
- iowrite32(value, addr) /* write-1-to-clear */.
- iounmap.
- for each set bit, log s5_reset_reason_txt[i].

REQ-29: print_dmi_agesa (late_initcall):
- dmi_walk(dmi_scan_additional, NULL).
- for each AGESA string in DMI_ENTRY_ADDITIONAL: pr_info("AGESA: %s").

REQ-30: rdmsrq_amd_safe / wrmsrq_amd_safe (K8-only):
- WARN_ONCE(boot_cpu_data.x86 != 0xf).
- gprs[7] = 0x9c5a203a.
- rdmsr_safe_regs / wrmsr_safe_regs.

REQ-31: rdrand_cmdline early_param: "force" ⟹ rdrand_force = true.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mem_encrypt_phys_bits_decrement_safe` | INVARIANT | per-early_detect_mem_encrypt: ((cpuid_ebx(0x8000001f) >> 6) & 0x3f) ≤ c.x86_phys_bits before subtract. |
| `va_align_mask_within_page_mask` | INVARIANT | per-bsp_init_amd fam 0x15: va_align.mask is subset of PAGE_MASK. |
| `dr_addr_mask_index_bounded` | INVARIANT | per-amd_set_dr_addr_mask: dr < 4. |
| `zen_generation_disjoint` | INVARIANT | per-bsp_init_amd: a given (fam, model) sets at most one of ZEN1..ZEN6. |
| `rdmsrq_amd_safe_k8_only` | INVARIANT | per-rdmsrq_amd_safe/wrmsrq_amd_safe: WARN_ONCE if boot_cpu_data.x86 != 0xf. |
| `tsa_init_zen34_microcode_match` | INVARIANT | per-tsa_init: ZEN3 ∨ ZEN4 ∧ microcode-match ⟹ VERW_CLEAR forced. |
| `mem_encrypt_clear_chain_complete` | INVARIANT | per-early_detect_mem_encrypt clear_all path: SME ∧ SEV ∧ SEV_ES ∧ SEV_SNP all cleared. |
| `s5_reset_w1c_only_when_not_u32max` | INVARIANT | per-print_s5_reset_status_mmio: iowrite32 happens only when value != U32_MAX. |

### Layer 2: TLA+

`arch/x86/cpu-amd.tla`:
- Per-CPU-init pipeline: pre-enum (early_init) → enum → post-enum (init) → bsp-only (bsp_init).
- Per-Zen-generation dispatch: bsp_init sets ZEN{1..6}; init dispatches init_amd_zen{1..5}.
- Per-mem-encrypt gate: early_detect_mem_encrypt has clear_all and clear_sev exits.
- Properties:
  - `safety_mem_encrypt_off_clears_all_three` — per-MSR_AMD64_SYSCFG.MEM_ENCRYPT clear ⟹ post-early_detect_mem_encrypt, SME ∧ SEV ∧ SEV_ES ∧ SEV_SNP cleared.
  - `safety_smmlock_clear_drops_sev_keeps_sme` — per-MSR_K7_HWCR.SMMLOCK clear with MEM_ENCRYPT set ⟹ SEV/SEV_ES/SEV_SNP cleared, SME retained.
  - `safety_snp_host_requires_zen34_or_rmpread` — per-bsp_determine_snp: CC_ATTR_HOST_SEV_SNP set ⟹ (ZEN3 ∨ ZEN4 ∨ RMPREAD) ∧ !HYPERVISOR.
  - `safety_zen_caps_disjoint_per_cpu` — per-CPU at most one of ZEN1..ZEN6 forced.
  - `safety_svm_bios_disabled_clears_svm` — per-init_amd: MSR_VM_CR.SVM_DIS ⟹ SVM cleared.
  - `safety_F16h_erratum793_msr_set` — per-early_init_amd: fam==0x16 ∧ model<=0xf ⟹ MSR_AMD64_LS_CFG.bit15 == 1.
  - `safety_zen1_div0_bug_set` — per-init_amd_zen1: X86_BUG_DIV0 set.
  - `safety_zen2_zenbleed_msr_consistent_with_ucode` — per-zen2_zenbleed_check: bit set ⟺ ucode does not match amd_zenbleed_microcode.
  - `liveness_per_AMD_CPU_init_terminates` — per-init_amd: returns in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Amd::early_init` post: K8 cap iff c.x86 >= 0xf | `Amd::early_init` |
| `Amd::early_init` post: TSC bits CONSTANT/NONSTOP/ACC_POWER/RAPL mirror c.x86_power | `Amd::early_init` |
| `Amd::early_detect_mem_encrypt` post: SEV cleared whenever MSR_K7_HWCR.SMMLOCK is clear | `Amd::early_detect_mem_encrypt` |
| `Amd::bsp_init` post: VA-align populated iff c.x86 == 0x15 | `Amd::bsp_init` |
| `Amd::bsp_init` post: Zen generation cap is at-most-one for any (fam, model) | `Amd::bsp_init` |
| `Amd::tsa_init` post: VERW_CLEAR forced iff ZEN3 ∨ ZEN4 with microcode match | `Amd::tsa_init` |
| `Amd::init` post: SVM cleared iff MSR_VM_CR.SVM_DIS | `Amd::init` |
| `Amd::init_k8` post: REP_GOOD iff CPUID(1).EAX in C+ stepping range | `Amd::init_k8` |
| `Amd::init_zen1` post: X86_BUG_DIV0 forced | `Amd::init_zen1` |
| `Amd::init_zen2` post: spectral chicken bit set iff !HYPERVISOR | `Amd::init_zen2` |
| `Amd::init_zen4` post: V_VMSAVE_VMLOAD cleared iff model ∈ {0x18..0x1f, 0x60..0x7f} | `Amd::init_zen4` |
| `Amd::detect_tlb` post: invlpgb_count_max = (cpuid_edx(0x80000008) & 0xffff) + 1 when INVLPGB | `Amd::detect_tlb` |
| `Amd::set_dr_addr_mask` post: WRMSR happens iff cached value != new mask | `Amd::set_dr_addr_mask` |

### Layer 4: Verus/Creusot functional

`Per-AMD-CPU-init: early_init_amd → identify_cpu (common) → init_amd (family dispatch + Zen dispatch) → bsp_init_amd (BSP only) → cpu_detect_tlb_amd` semantic equivalence: per-AMD PPR / BKDG / Open-Source Register Reference for the relevant families (K7, K8, Fam10h, Fam12h, Fam15h, Fam16h, Fam17h Zen, Fam19h Zen3/4, Fam1Ah Zen5/6) and per-Documentation/x86/amd-memory-encryption.rst for SME/SEV/SEV_SNP gating.

### hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening and `cpu-common.md`.)

AMD-cpu reinforcement:

- **Per-MSR_AMD64_SYSCFG.MEM_ENCRYPT gate** — defense against per-SME/SEV-claimed-but-BIOS-disabled state leaking encryption assumptions into mm.
- **Per-MSR_K7_HWCR.SMMLOCK gate before SEV** — defense against per-untrusted-SMM SEV attack surface.
- **Per-bsp_determine_snp HYPERVISOR exclusion + RMP table probe** — defense against guest-side spurious SNP-host claim.
- **Per-fam-16h LS_CFG bit 15 (erratum 793, CVE-2013-6885)** — defense against per-FXSAVE-pointer-leak side channel.
- **Per-fam-15..17 LS_CFG-bit SSBD synthesis** — defense against Spectre-v4 (Speculative Store Bypass).
- **Per-Zen2 spectral-chicken MSR_ZEN2_SPECTRAL_CHICKEN** — defense against mid-basic-block speculation.
- **Per-Zen2 zenbleed MSR_AMD64_DE_CFG.FP_BACKUP_FIX** — defense against per-XMM-register cross-context leak.
- **Per-Zen1 DIV0 + IRPERF (erratum 1054) + FPDSS denorm fix** — defense against per-CPU-numerical-correctness failure and timing-side-channel via IRPERF.
- **Per-Zen4 shared-BTB-fix and V_VMSAVE_VMLOAD clear on specific models** — defense against host-reboot-via-malicious-VMSAVE and BTB cross-context leakage.
- **Per-Zen5 RDSEED32 microcode gate** — defense against per-broken-CSRNG entropy use.
- **Per-init_amd_k8 erratum #110 LAHF_LM clear** — defense against per-BIOS-falsely-set-feature use of unsupported instruction.
- **Per-init_amd_gh GART TLB-Walk-Errors + WC+ corruption** — defense against per-fam10h MCE flood and per-WC+ memory-type miscoercion.
- **Per-K8 rdmsrq_amd_safe / wrmsrq_amd_safe with WARN_ONCE family guard** — defense against per-cross-family MSR-encoded password leakage / GP.
- **Per-AutoIBRS guarantee via MSR_EFER._EFER_AUTOIBRS** — defense against per-AP-bring-up missing eIBRS replication.
- **Per-amd_set_dr_addr_mask BPEXT cap guard + dr<4 WARN** — defense against per-BPEXT-disabled MSR access and per-out-of-bounds index.
- **Per-clear_rdrand_cpuid_bit on BIOS-broken-suspend** — defense against per-RDRAND returning stale entropy after S3.
- **Per-invlpgb_count_max strict per CPUID** — defense against per-INVLPGB over-flush DoS.
- **Per-amd_check_microcode ZEN2 broadcast** — defense against per-CPU-isolated zenbleed mitigation drift after late ucode load.

