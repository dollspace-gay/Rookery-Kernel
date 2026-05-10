# Tier-3: arch/x86/kvm/cpuid.c — KVM CPUID emulation per vCPU

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: arch/x86/00-overview.md
upstream-paths:
  - arch/x86/kvm/cpuid.c (~2177 lines)
  - arch/x86/kvm/cpuid.h
  - arch/x86/kvm/reverse_cpuid.h
  - arch/x86/include/uapi/asm/kvm.h (struct kvm_cpuid, kvm_cpuid2, kvm_cpuid_entry, kvm_cpuid_entry2)
  - arch/x86/include/uapi/asm/kvm_para.h (KVM_FEATURE_*, KVM_CPUID_FEATURES, KVM_CPUID_SIGNATURE)
  - include/uapi/linux/kvm.h (KVM_SET_CPUID, KVM_SET_CPUID2, KVM_GET_CPUID2, KVM_GET_SUPPORTED_CPUID, KVM_GET_EMULATED_CPUID)
-->

## Summary

KVM emulates the x86 **CPUID** instruction inside each guest vCPU. Userspace VMM (QEMU, cloud-hypervisor, crosvm) builds a per-vCPU CPUID table — an array of `struct kvm_cpuid_entry2` — and installs it via the `KVM_SET_CPUID2` ioctl; KVM stores it in `vcpu->arch.cpuid_entries` (with `cpuid_nent` count). When the guest executes `CPUID`, the VMX/SVM exit handler calls `kvm_emulate_cpuid()` → `kvm_cpuid()` → `kvm_find_cpuid_entry_index()` to look up the table by `(function, index)` and writes the entry's `eax/ebx/ecx/edx` back to the guest's GPRs. A global `kvm_cpu_caps[]` bitmap, initialized once by `kvm_initialize_cpu_caps()`, narrows what KVM permits userspace to expose. A runtime layer (`kvm_update_cpuid_runtime`) keeps dynamic bits (OSXSAVE, OSPKE, APIC-enable, MWAIT, XSAVE area sizes) in sync with current vCPU state. KVM also synthesizes a Hyper-V-shaped hypervisor leaf cluster (when CONFIG_KVM_HYPERV) and the KVM paravirt-feature leaf `KVM_CPUID_FEATURES` (`0x40000001`) advertising PV-clock, async-PF, PV-EOI, PV-unhalt, PV-TLB-flush, PV-send-IPI, PV-sched-yield, steal-time, poll-control, and async-PF-INT. Critical for: faithful x86 CPU identification, gating of guest instruction-set use, paravirt feature negotiation, and post-CPUID side effects (MMU MAXPHYADDR, LAPIC TSC-deadline mode, FPU guest-XFD enable, reserved-CR4 bits).

This Tier-3 covers `arch/x86/kvm/cpuid.c` (~2177 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_cpuid_entry2` | per-leaf record (function, index, flags, eax-edx) | `KvmCpuidEntry2` |
| `struct kvm_cpuid2` | per-ioctl header (nent + entries[]) | `KvmCpuid2` |
| `kvm_cpu_caps[]` (NR_KVM_CPU_CAPS) | per-feature global mask | `Kvm::cpu_caps` |
| `kvm_initialize_cpu_caps()` | per-module init of `kvm_cpu_caps` | `Kvm::initialize_cpu_caps` |
| `kvm_find_cpuid_entry2()` | per-(function,index) lookup in array | `KvmCpuid::find_entry2` |
| `kvm_check_cpuid()` | per-vCPU validation post-SET | `KvmCpuid::check_cpuid` |
| `kvm_cpuid_check_equal()` | per-vCPU no-change verify (post-RUN) | `KvmCpuid::check_equal` |
| `kvm_set_cpuid()` | per-vCPU install | `KvmCpuid::set_cpuid` |
| `kvm_vcpu_ioctl_set_cpuid()` | per-`KVM_SET_CPUID` ioctl | `KvmCpuid::ioctl_set_cpuid` |
| `kvm_vcpu_ioctl_set_cpuid2()` | per-`KVM_SET_CPUID2` ioctl | `KvmCpuid::ioctl_set_cpuid2` |
| `kvm_vcpu_ioctl_get_cpuid2()` | per-`KVM_GET_CPUID2` ioctl | `KvmCpuid::ioctl_get_cpuid2` |
| `kvm_dev_ioctl_get_cpuid()` | per-`KVM_GET_SUPPORTED_CPUID` / `KVM_GET_EMULATED_CPUID` ioctl | `KvmCpuid::dev_ioctl_get_cpuid` |
| `kvm_update_cpuid_runtime()` | per-dynamic-bit refresh | `KvmCpuid::update_runtime` |
| `kvm_vcpu_after_set_cpuid()` | per-vCPU post-install side-effects | `KvmCpuid::after_set_cpuid` |
| `kvm_apply_cpuid_pv_features_quirk()` | per-vCPU PV-feature reconciliation | `KvmCpuid::apply_pv_features_quirk` |
| `kvm_get_hypervisor_cpuid()` | per-vCPU hypervisor-base discovery | `KvmCpuid::get_hypervisor_cpuid` |
| `kvm_cpuid_has_hyperv()` | per-vCPU Hyper-V detection | `KvmCpuid::has_hyperv` |
| `cpuid_func_emulated()` | per-function emulation table | `KvmCpuid::func_emulated` |
| `__do_cpuid_func()` | per-host-CPUID + leaf-rewrite | `KvmCpuid::do_func` |
| `do_host_cpuid()` | per-leaf raw read + array append | `KvmCpuid::do_host_cpuid` |
| `get_out_of_range_cpuid_entry()` | per-out-of-range max-basic-redirect | `KvmCpuid::out_of_range_entry` |
| `kvm_cpuid()` | per-CPUID-emulation evaluator | `KvmCpuid::cpuid` |
| `kvm_emulate_cpuid()` | per-CPUID-VM-exit handler | `KvmCpuid::emulate` |
| `cpuid_query_maxphyaddr()` | per-vCPU MAXPHYADDR (CPUID.80000008:EAX[7:0]) | `KvmCpuid::query_maxphyaddr` |
| `cpuid_query_maxguestphyaddr()` | per-vCPU guest-MAXPHYADDR (EAX[23:16]) | `KvmCpuid::query_maxguestphyaddr` |
| `kvm_vcpu_reserved_gpa_bits_raw()` | per-vCPU rsvd-GPA-bit mask | `KvmCpuid::reserved_gpa_bits_raw` |
| `xstate_required_size()` | per-XCR0 XSAVE area size compute | `KvmCpuid::xstate_required_size` |
| `kvm_init_xstate_sizes()` | per-module XSAVE-area-table init | `KvmCpuid::init_xstate_sizes` |
| `KVM_FEATURE_*` bits | PV-feature enumeration (CLOCKSOURCE, NOP_IO_DELAY, CLOCKSOURCE2, ASYNC_PF, PV_EOI, PV_UNHALT, PV_TLB_FLUSH, ASYNC_PF_VMEXIT, PV_SEND_IPI, POLL_CONTROL, PV_SCHED_YIELD, ASYNC_PF_INT, STEAL_TIME, CLOCKSOURCE_STABLE_BIT) | shared |

## Compatibility contract

REQ-1: struct kvm_cpuid_entry2 (UAPI; bitwise identical):
- function: u32, CPUID leaf number.
- index: u32, CPUID sub-leaf (only meaningful when flags & `KVM_CPUID_FLAG_SIGNIFCANT_INDEX`).
- flags: u32, bitmask — `KVM_CPUID_FLAG_SIGNIFCANT_INDEX`, `KVM_CPUID_FLAG_STATEFUL_FUNC`, `KVM_CPUID_FLAG_STATE_READ_NEXT`.
- eax, ebx, ecx, edx: u32, CPUID output registers.
- padding[3]: u32, must be zero (validated for `KVM_GET_EMULATED_CPUID`).

REQ-2: struct kvm_cpuid2 (UAPI):
- nent: u32 in, count of entries.
- padding: u32 (reserved 0).
- entries[]: flexible array of `struct kvm_cpuid_entry2`.

REQ-3: KVM_SET_CPUID2 ioctl (`kvm_vcpu_ioctl_set_cpuid2`):
- if cpuid.nent > KVM_MAX_CPUID_ENTRIES (256): return -E2BIG.
- e2 = vmemdup_array_user(entries, nent).
- if IS_ERR(e2): return PTR_ERR(e2).
- r = kvm_set_cpuid(vcpu, e2, cpuid->nent).
- on err: kvfree(e2).

REQ-4: kvm_set_cpuid(vcpu, e2, nent):
- if vcpu.arch.cpuid_dynamic_bits_dirty: kvm_update_cpuid_runtime(vcpu).
- swap(vcpu.arch.cpuid_entries, e2); swap(vcpu.arch.cpuid_nent, nent).
- memcpy(vcpu_caps, vcpu.arch.cpu_caps).
- if !kvm_can_set_cpuid_and_feature_msrs(vcpu):
  - r = kvm_cpuid_check_equal(vcpu, e2, nent); on err goto err; goto success.
- if CONFIG_KVM_HYPERV ∧ kvm_cpuid_has_hyperv(vcpu): r = kvm_hv_vcpu_init; on err goto err.
- r = kvm_check_cpuid(vcpu); on err goto err.
- if CONFIG_KVM_XEN: vcpu.arch.xen.cpuid = kvm_get_hypervisor_cpuid(vcpu, XEN_SIGNATURE).
- kvm_vcpu_after_set_cpuid(vcpu).
- success: kvfree(e2); return 0.
- err: restore cpu_caps; swap back cpuid_entries, cpuid_nent; return r.

REQ-5: kvm_check_cpuid(vcpu):
- best = kvm_find_cpuid_entry(vcpu, 0x80000008).
- if best: vaddr_bits = (best.eax & 0xff00) >> 8; if vaddr_bits ∉ {0, 48, 57}: return -EINVAL.
- best = kvm_find_cpuid_entry_index(vcpu, 0xd, 0).
- if !best: return 0.
- xfeatures = (best.eax | ((u64)best.edx << 32)) & XFEATURE_MASK_USER_DYNAMIC.
- if !xfeatures: return 0.
- return fpu_enable_guest_xfd_features(&vcpu.arch.guest_fpu, xfeatures).

REQ-6: kvm_cpuid_check_equal (post-RUN no-change verification):
- kvm_update_cpuid_runtime(vcpu); kvm_apply_cpuid_pv_features_quirk(vcpu).
- if nent != vcpu.arch.cpuid_nent: return -EINVAL.
- for i in 0..nent: if any of function, index, flags, eax, ebx, ecx, edx differ from `vcpu.arch.cpuid_entries[i]`: return -EINVAL.
- return 0.

REQ-7: kvm_find_cpuid_entry2(entries, nent, function, index):
- lockdep_assert_irqs_enabled (IRQ-disabled CPUID lookup is forbidden by KVM's perf rule).
- for i in 0..nent:
  - e = &entries[i]. if e.function != function: continue.
  - if !(e.flags & SIGNIFCANT_INDEX) ∨ e.index == index: return e.
  - if index == KVM_CPUID_INDEX_NOT_SIGNIFICANT: WARN if architecturally indexed; return e.
- return NULL.

REQ-8: kvm_update_cpuid_runtime(vcpu) — keep dynamic bits in sync:
- vcpu.arch.cpuid_dynamic_bits_dirty = false.
- leaf-1: X86_FEATURE_OSXSAVE = CR4.OSXSAVE; X86_FEATURE_APIC = APIC_BASE.EN; X86_FEATURE_MWAIT = MISC_ENABLE.MWAIT (unless MISC_ENABLE_NO_MWAIT quirk).
- leaf-7.0: X86_FEATURE_OSPKE = CR4.PKE.
- leaf-0xD.0: best.ebx = xstate_required_size(xcr0, false).
- leaf-0xD.1 (if XSAVES|XSAVEC): best.ebx = xstate_required_size(xcr0 | ia32_xss, true).

REQ-9: kvm_vcpu_after_set_cpuid(vcpu) — post-install side effects:
- memset(vcpu.arch.cpu_caps, 0).
- for i in 0..NR_KVM_CPU_CAPS: per-`reverse_cpuid[i]` map, vcpu.cpu_caps[i] = (kvm_cpu_caps[i] | emulated[i]) & guest_entry_reg.
- kvm_update_cpuid_runtime(vcpu).
- allow_gbpages = tdp_enabled ? host-has-GBPAGES : guest-has-GBPAGES; guest_cpu_cap_change(GBPAGES).
- leaf-1 ∧ apic: TSC_DEADLINE_TIMER → lapic_timer.timer_mode_mask = 3<<17 else 1<<17; kvm_apic_set_version.
- vcpu.arch.guest_supported_xcr0 = cpuid_get_supported_xcr0.
- vcpu.arch.guest_supported_xss = cpuid_get_supported_xss.
- vcpu.arch.pv_cpuid.features = kvm_apply_cpuid_pv_features_quirk.
- vcpu.arch.is_amd_compatible = guest_cpuid_is_amd_or_hygon.
- vcpu.arch.maxphyaddr = cpuid_query_maxphyaddr.
- vcpu.arch.reserved_gpa_bits = kvm_vcpu_reserved_gpa_bits_raw.
- kvm_pmu_refresh(vcpu).
- vcpu.arch.cr4_guest_rsvd_bits = __cr4_reserved_bits(kvm) | __cr4_reserved_bits(guest).
- kvm_hv_set_cpuid(vcpu, has_hyperv).
- kvm_x86_call(vcpu_after_set_cpuid)(vcpu) (vendor: VMX/SVM).
- kvm_mmu_after_set_cpuid(vcpu).
- kvm_make_request(KVM_REQ_RECALC_INTERCEPTS, vcpu).

REQ-10: kvm_apply_cpuid_pv_features_quirk(vcpu):
- kvm_cpuid = kvm_get_hypervisor_cpuid(vcpu, KVM_SIGNATURE).
- if !kvm_cpuid.base: return 0.
- best = kvm_find_cpuid_entry(vcpu, kvm_cpuid.base | KVM_CPUID_FEATURES).
- if !best: return 0.
- if kvm_hlt_in_guest(vcpu.kvm): best.eax &= ~(1 << KVM_FEATURE_PV_UNHALT).
- return best.eax.

REQ-11: kvm_initialize_cpu_caps() — per-module init of `kvm_cpu_caps[NR_KVM_CPU_CAPS]`:
- For each kernel-defined feature leaf, mask raw host CPUID against KVM's supported feature set:
  - CPUID_1_EDX/ECX, CPUID_7_0_EBX/ECX/EDX, CPUID_7_1_EAX/ECX/EDX, CPUID_7_2_EDX,
  - CPUID_D_1_EAX, CPUID_8000_0001_EDX/ECX, CPUID_8000_0007_EDX, CPUID_8000_0008_EBX,
  - CPUID_8000_000A_EDX, CPUID_8000_001F_EAX, CPUID_8000_0021_EAX/ECX, CPUID_8000_0022_EAX,
  - CPUID_C000_0001_EDX, etc.
- kvm_is_configuring_cpu_caps = true during init; false after.

REQ-12: kvm_dev_ioctl_get_cpuid (KVM_GET_SUPPORTED_CPUID / KVM_GET_EMULATED_CPUID):
- funcs = { 0x0, 0x80000000, 0xC0000000, KVM_CPUID_SIGNATURE (0x40000000) }.
- for each func: get_cpuid_func(array, func, type) which walks the leaf range up to entries[nent-1].eax.
- KVM_CPUID_SIGNATURE (0x40000000) entry holds KVM_SIGNATURE ("KVMKVMKVM\0\0\0") in ebx/ecx/edx and limit = 0x40000001 in eax.
- KVM_CPUID_FEATURES (0x40000001) advertises:
  - KVM_FEATURE_CLOCKSOURCE, KVM_FEATURE_NOP_IO_DELAY, KVM_FEATURE_CLOCKSOURCE2,
  - KVM_FEATURE_ASYNC_PF, KVM_FEATURE_PV_EOI, KVM_FEATURE_CLOCKSOURCE_STABLE_BIT,
  - KVM_FEATURE_PV_UNHALT, KVM_FEATURE_PV_TLB_FLUSH, KVM_FEATURE_ASYNC_PF_VMEXIT,
  - KVM_FEATURE_PV_SEND_IPI, KVM_FEATURE_POLL_CONTROL, KVM_FEATURE_PV_SCHED_YIELD,
  - KVM_FEATURE_ASYNC_PF_INT, KVM_FEATURE_STEAL_TIME (if sched_info_on()).

REQ-13: __do_cpuid_func(array, function) — per-leaf canonicalization:
- get_cpu(); entry = do_host_cpuid(array, function, 0).
- case 0x0: entry.eax = min(entry.eax, 0x24).
- case 0x1: cpuid_entry_override(CPUID_1_EDX, CPUID_1_ECX).
- case 0x4 / 0x8000001d: iterate index until cache type (entry.eax & 0x1f) == 0.
- case 0x6: thermal = ARAT only (entry.eax = 0x4).
- case 0x7: cap max_idx ≤ 2; override CPUID_7_0_EBX/ECX/EDX, 7_1_EAX/ECX/EDX, 7_2_EDX.
- case 0xa (Architectural PMU): synthesize from `kvm_pmu_cap` (version, num_counters_gp, bit_width_gp, events_mask_len; fixed counters; anythread_deprecated).
- case 0xb / 0x1f (topology): zero (presence-of-subleaf-1 indicates valid topology).
- case 0xd (XSAVE): eax &= permitted_xcr0; ebx = xstate_required_size; iterate index 2..63 mapping to XCR0/XSS bits.
- case 0x80000000: eax = min(eax, KVM-supported max extended leaf).
- case 0x80000008: phys_as / virt_as / g_phys_as packing depends on tdp_enabled.
- case 0x8000000A (SVM): synthesize when SVM-supported.
- case 0x8000001F (SEV): clear NumVMPL; mask PA-bits-reduction.
- case 0x80000021 (AMD): cpuid_entry_override(CPUID_8000_0021_EAX/ECX); ERAPS handling.
- case 0x80000022 (AMD PMU v2): synthesize from kvm_pmu_cap.
- case 0xC0000000..0xC0000004 (Centaur/Zhaoxin): cap eax = 0xC0000004; CPUID_C000_0001_EDX override.
- default: zero output.

REQ-14: kvm_cpuid(vcpu, &eax, &ebx, &ecx, &edx, exact_only) — emulation evaluator:
- if vcpu.arch.cpuid_dynamic_bits_dirty: kvm_update_cpuid_runtime(vcpu).
- entry = kvm_find_cpuid_entry_index(vcpu, function, index).
- exact = (entry != NULL).
- if !entry ∧ !exact_only: entry = get_out_of_range_cpuid_entry(vcpu, &function, index); used_max_basic = !!entry.
- if entry: copy eax-edx out; per-function fixups (leaf 7.0 TSX-CTRL clearing of RTM/HLE; leaf 0x80000007 CONSTANT_TSC if Hyper-V invtsc suppressed; Xen TSC leaves).
- else: zero out; leaf 0xb / 0x1f pass-through CL = index & 0xff and EDX = x2APIC ID from subleaf 1.
- trace_kvm_cpuid(orig_function, index, *eax, *ebx, *ecx, *edx, exact, used_max_basic).
- return exact.

REQ-15: kvm_emulate_cpuid(vcpu) — VM-exit handler:
- if !is_smm(vcpu) ∧ cpuid_fault_enabled(vcpu) ∧ !kvm_require_cpl(vcpu, 0): return 1 (inject #GP).
- eax = kvm_rax_read; ecx = kvm_rcx_read.
- kvm_cpuid(vcpu, &eax, &ebx, &ecx, &edx, false).
- kvm_rax_write; kvm_rbx_write; kvm_rcx_write; kvm_rdx_write.
- return kvm_skip_emulated_instruction(vcpu).

REQ-16: get_out_of_range_cpuid_entry — Intel-style max-basic redirect:
- basic = leaf 0; if vendor is AMD/Hygon: return NULL (AMD does not redirect).
- class lookup based on function range (0x40000000..0x4fffffff = function & 0xffffff00; 0xc0000000+ = 0xc0000000; else = function & 0x80000000).
- if class ∧ function ≤ class.eax: return NULL.
- *fn_ptr = basic.eax (rewrite to max-basic-leaf).
- return kvm_find_cpuid_entry_index(vcpu, basic.eax, index).

REQ-17: Hyper-V signature detection (CONFIG_KVM_HYPERV):
- kvm_cpuid_has_hyperv(vcpu): entry = kvm_find_cpuid_entry(HYPERV_CPUID_INTERFACE); return entry ∧ entry.eax == HYPERV_CPUID_SIGNATURE_EAX ("Hv#1").
- post-set: kvm_hv_set_cpuid(vcpu, has_hyperv) wires Hyper-V-only enlightenments.

REQ-18: KVM_GET_EMULATED_CPUID strict padding check (`sanity_check_entries`):
- For each entry: copy_from_user(padding[3]); if any padding word != 0: reject.

## Acceptance Criteria

- [ ] AC-1: `KVM_SET_CPUID2` with `nent > KVM_MAX_CPUID_ENTRIES` → -E2BIG.
- [ ] AC-2: `KVM_SET_CPUID2` then `KVM_GET_CPUID2` round-trips byte-identical entries (modulo runtime-dynamic bits OSXSAVE, OSPKE, APIC, MWAIT, XSAVE-size).
- [ ] AC-3: Guest `CPUID(0x40000000)` returns KVM signature "KVMKVMKVM\0\0\0" in EBX/ECX/EDX with EAX = 0x40000001.
- [ ] AC-4: Guest `CPUID(0x40000001)` advertises configured `KVM_FEATURE_*` bits in EAX.
- [ ] AC-5: Guest `CPUID(0)` with leaf > guest-max-basic returns the max-basic leaf data (Intel-style redirect); AMD/Hygon-vendored CPUID returns zero.
- [ ] AC-6: After `kvm_set_cpuid`, `kvm_check_cpuid` rejects vaddr_bits ∉ {0, 48, 57} in CPUID.0x80000008:EAX[15:8].
- [ ] AC-7: After KVM_RUN, a subsequent `KVM_SET_CPUID2` with different entries → -EINVAL via `kvm_cpuid_check_equal`.
- [ ] AC-8: Setting CR4.OSXSAVE causes CPUID.1:ECX bit 27 (OSXSAVE) to flip without a new SET_CPUID call (dynamic-bit refresh).
- [ ] AC-9: `kvm_hlt_in_guest(kvm)` strips `KVM_FEATURE_PV_UNHALT` from CPUID.0x40000001:EAX (PV-features quirk).
- [ ] AC-10: CPUID-fault MSR enabled ∧ guest CPL>0 ∧ !SMM → CPUID raises #GP (return 1 from `kvm_emulate_cpuid`).
- [ ] AC-11: Lookup of CPUID with IRQs disabled triggers `lockdep_assert_irqs_enabled` (debug-only, no functional effect).
- [ ] AC-12: `KVM_GET_EMULATED_CPUID` rejects any entry with non-zero padding via `sanity_check_entries`.
- [ ] AC-13: TSC_DEADLINE_TIMER set in CPUID.1:ECX → `lapic_timer.timer_mode_mask == 3 << 17` after `kvm_vcpu_after_set_cpuid`.
- [ ] AC-14: TDP enabled → CPUID.0x80000008:EAX[7:0] (phys_as) follows guest entry; TDP disabled → phys_as forced to host `x86_phys_bits`.
- [ ] AC-15: `kvm_cpu_caps` mask applied — guest cannot advertise an unsupported feature even if userspace sets the bit.

## Architecture

```
struct KvmCpuidEntry2 {
  function: u32,
  index:    u32,
  flags:    u32,             // KVM_CPUID_FLAG_{SIGNIFCANT_INDEX, STATEFUL_FUNC, STATE_READ_NEXT}
  eax:      u32,
  ebx:      u32,
  ecx:      u32,
  edx:      u32,
  padding:  [u32; 3],        // must be zero on GET_EMULATED_CPUID
}

struct KvmCpuid2 {
  nent:     u32,             // input/output count
  padding:  u32,
  entries:  [KvmCpuidEntry2], // flexible array (nent elements)
}

struct VcpuCpuidState {
  cpuid_entries:             Vec<KvmCpuidEntry2>,   // owned per-vCPU
  cpuid_nent:                u32,
  cpu_caps:                  [u32; NR_KVM_CPU_CAPS], // guest cap mask (intersection)
  cpuid_dynamic_bits_dirty:  bool,
  guest_supported_xcr0:      u64,
  guest_supported_xss:       u64,
  pv_cpuid_features:         u32,
  is_amd_compatible:         bool,
  maxphyaddr:                u32,
  reserved_gpa_bits:         u64,
  cr4_guest_rsvd_bits:       u64,
}

static KVM_CPU_CAPS: [u32; NR_KVM_CPU_CAPS]; // global, set by initialize_cpu_caps
```

`KvmCpuid::ioctl_set_cpuid2(vcpu, cpuid, entries_user) -> Result<(), errno>`:
1. if cpuid.nent > KVM_MAX_CPUID_ENTRIES: return -E2BIG.
2. if cpuid.nent > 0: e2 = vmemdup_array_user(entries_user, cpuid.nent)?.
3. r = `KvmCpuid::set_cpuid(vcpu, e2, cpuid.nent)`.
4. if r != 0: kvfree(e2).
5. return r.

`KvmCpuid::set_cpuid(vcpu, e2, nent) -> Result<(), errno>`:
1. if vcpu.arch.cpuid_dynamic_bits_dirty: `KvmCpuid::update_runtime(vcpu)`.
2. swap(vcpu.arch.cpuid_entries, e2); swap(vcpu.arch.cpuid_nent, nent).
3. memcpy(vcpu_caps, vcpu.arch.cpu_caps).
4. if !kvm_can_set_cpuid_and_feature_msrs(vcpu):
   - r = `KvmCpuid::check_equal(vcpu, e2, nent)`.
   - if r != 0: goto err.
   - goto success.
5. if cfg!(CONFIG_KVM_HYPERV) ∧ `KvmCpuid::has_hyperv(vcpu)`: r = kvm_hv_vcpu_init; if r: goto err.
6. r = `KvmCpuid::check_cpuid(vcpu)`; if r: goto err.
7. if cfg!(CONFIG_KVM_XEN): vcpu.arch.xen.cpuid = `KvmCpuid::get_hypervisor_cpuid(vcpu, XEN_SIGNATURE)`.
8. `KvmCpuid::after_set_cpuid(vcpu)`.
9. success: kvfree(e2); return 0.
10. err: restore cpu_caps; swap back cpuid_entries, cpuid_nent; return r.

`KvmCpuid::find_entry2(entries, nent, function, index) -> Option<&KvmCpuidEntry2>`:
1. lockdep_assert_irqs_enabled.
2. for e in entries[..nent]:
   - if e.function != function: continue.
   - if !(e.flags & KVM_CPUID_FLAG_SIGNIFCANT_INDEX) ∨ e.index == index: return Some(e).
   - if index == KVM_CPUID_INDEX_NOT_SIGNIFICANT:
     - WARN_ON_ONCE(cpuid_function_is_indexed(function)).
     - return Some(e).
3. return None.

`KvmCpuid::cpuid(vcpu, eax, ebx, ecx, edx, exact_only) -> bool`:
1. orig_function = *eax; function = *eax; index = *ecx.
2. if vcpu.arch.cpuid_dynamic_bits_dirty: `KvmCpuid::update_runtime(vcpu)`.
3. entry = `KvmCpuid::find_entry2(.., function, index)`.
4. exact = entry.is_some().
5. if entry.is_none() ∧ !exact_only:
   - entry = `KvmCpuid::out_of_range_entry(vcpu, &mut function, index)`.
   - used_max_basic = entry.is_some().
6. if entry.is_some(): copy eax-edx; per-function fixup (leaf 7.0 TSX_CTRL.CPUID_CLEAR → strip RTM|HLE; 0x80000007 → strip CONSTANT_TSC if HV invtsc suppressed; Xen TSC leaf → fill pvclock_tsc_mul/shift, hw_tsc_khz).
7. else: zero out; leaf 0xb / 0x1f → CL pass-through, EDX from subleaf 1.
8. trace_kvm_cpuid(orig_function, index, ..., exact, used_max_basic).
9. return exact.

`KvmCpuid::emulate(vcpu) -> u32`:
1. if !is_smm(vcpu) ∧ cpuid_fault_enabled(vcpu) ∧ !kvm_require_cpl(vcpu, 0): return 1.
2. eax = kvm_rax_read(vcpu); ecx = kvm_rcx_read(vcpu); ebx = 0; edx = 0.
3. `KvmCpuid::cpuid(vcpu, &mut eax, &mut ebx, &mut ecx, &mut edx, false)`.
4. kvm_rax_write(vcpu, eax); kvm_rbx_write(vcpu, ebx); kvm_rcx_write(vcpu, ecx); kvm_rdx_write(vcpu, edx).
5. return kvm_skip_emulated_instruction(vcpu).

`KvmCpuid::update_runtime(vcpu)`:
1. vcpu.arch.cpuid_dynamic_bits_dirty = false.
2. best = find leaf 1 → OSXSAVE = CR4.OSXSAVE; APIC = (APIC_BASE & MSR_IA32_APICBASE_ENABLE); MWAIT (if !MISC_ENABLE_NO_MWAIT quirk) = MISC_ENABLE.MWAIT.
3. best = find leaf 7.0 → OSPKE = CR4.PKE.
4. best = find leaf 0xD.0 → ebx = `xstate_required_size(xcr0, false)`.
5. best = find leaf 0xD.1 (if XSAVES|XSAVEC) → ebx = `xstate_required_size(xcr0 | ia32_xss, true)`.

`KvmCpuid::after_set_cpuid(vcpu)`:
1. memset(cpu_caps, 0); for i in 0..NR_KVM_CPU_CAPS map reverse_cpuid[i] → cpu_caps[i] = (KVM_CPU_CAPS[i] | emulated_reg(i)) & guest_reg(i).
2. `KvmCpuid::update_runtime(vcpu)`.
3. allow_gbpages = tdp_enabled ? boot_cpu_has(GBPAGES) : guest_cpu_cap_has(GBPAGES); guest_cpu_cap_change(GBPAGES, allow_gbpages).
4. leaf 1 ∧ apic: lapic_timer.timer_mode_mask = TSC_DEADLINE_TIMER ? 3<<17 : 1<<17; kvm_apic_set_version.
5. guest_supported_xcr0 = (best_0xD_0.eax | (edx << 32)) & kvm_caps.supported_xcr0.
6. guest_supported_xss = (best_0xD_1.ecx | (edx << 32)) & kvm_caps.supported_xss.
7. pv_cpuid.features = `KvmCpuid::apply_pv_features_quirk(vcpu)`.
8. is_amd_compatible = (leaf 0 vendor == AMD ∨ HYGON).
9. maxphyaddr = `KvmCpuid::query_maxphyaddr(vcpu)`.
10. reserved_gpa_bits = `KvmCpuid::reserved_gpa_bits_raw(vcpu)`.
11. kvm_pmu_refresh(vcpu).
12. cr4_guest_rsvd_bits = __cr4_reserved_bits(kvm_cpu_cap_has) | __cr4_reserved_bits(guest_cpu_cap_has).
13. kvm_hv_set_cpuid(vcpu, `KvmCpuid::has_hyperv(vcpu)`).
14. kvm_x86_call(vcpu_after_set_cpuid)(vcpu).
15. kvm_mmu_after_set_cpuid(vcpu).
16. kvm_make_request(KVM_REQ_RECALC_INTERCEPTS, vcpu).

`KvmCpuid::out_of_range_entry(vcpu, fn_ptr, index) -> Option<&KvmCpuidEntry2>`:
1. basic = find leaf 0; if !basic: return None.
2. if vendor is AMD ∨ HYGON (per is_guest_vendor_*): return None.
3. class = leaf-class lookup (0x4xxxxxxx → fn & 0xffffff00; ≥0xc0000000 → 0xc0000000; else → fn & 0x80000000).
4. if class ∧ function ≤ class.eax: return None.
5. *fn_ptr = basic.eax (rewrite caller's function pointer to max-basic).
6. return `KvmCpuid::find_entry2(.., basic.eax, index)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `entries_nent_in_bounds` | INVARIANT | per-find_entry2: index `i` ∈ [0, nent). |
| `set_cpuid_nent_capped` | INVARIANT | per-ioctl_set_cpuid2: cpuid.nent ≤ KVM_MAX_CPUID_ENTRIES. |
| `swap_restore_on_error` | INVARIANT | per-set_cpuid: on error path, cpuid_entries and cpuid_nent swapped back. |
| `padding_zero_strict` | INVARIANT | per-GET_EMULATED_CPUID: all padding bytes zero. |
| `dynamic_bits_dirty_reset_after_update` | INVARIANT | per-update_runtime: cpuid_dynamic_bits_dirty == false on exit. |
| `find_entry2_irqs_enabled` | INVARIANT | per-find_entry2: lockdep_assert_irqs_enabled. |
| `cpu_caps_intersection` | INVARIANT | per-after_set_cpuid: cpu_caps[i] ⊆ kvm_cpu_caps[i] (no superset). |

### Layer 2: TLA+

`arch/x86/kvm/cpuid.tla`:
- Per-ioctl-SET → per-validate → per-runtime-update → per-side-effects → per-GET round-trip.
- Per-emulate: per-find → per-out-of-range-redirect → per-fixup → per-trace.
- Properties:
  - `safety_set_get_round_trip` — per-`KVM_SET_CPUID2`, immediate `KVM_GET_CPUID2` returns same entries (modulo dynamic bits).
  - `safety_invalid_set_preserves_state` — per-`kvm_check_cpuid` failure: vcpu.cpuid_entries unchanged.
  - `safety_post_run_set_must_be_equal` — per-RUN-then-SET: any byte-diff → -EINVAL.
  - `safety_cpu_caps_mask_enforced` — per-emulate: guest cannot see a bit not in `kvm_cpu_caps`.
  - `safety_hlt_in_guest_strips_pv_unhalt` — per-PV-quirk: `KVM_FEATURE_PV_UNHALT` cleared when guest-HLT enabled.
  - `safety_amd_no_max_basic_redirect` — per-out-of-range: AMD/Hygon vendor → NULL.
  - `liveness_emulate_terminates` — per-emulate_cpuid: terminates and advances RIP.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `KvmCpuid::set_cpuid` post: ret == 0 ⟹ vcpu.cpuid_entries == e2-input | `KvmCpuid::set_cpuid` |
| `KvmCpuid::set_cpuid` err-post: state == prior-state | `KvmCpuid::set_cpuid` |
| `KvmCpuid::find_entry2` post: returns Some(e) ⟹ e.function == function ∧ (e.index == index ∨ !significant) | `KvmCpuid::find_entry2` |
| `KvmCpuid::cpuid` post: exact_only ∧ no-entry ⟹ outputs unchanged from input pointers (caller responsible) | `KvmCpuid::cpuid` |
| `KvmCpuid::emulate` post: GPRs updated ∧ RIP advanced ∨ #GP injected | `KvmCpuid::emulate` |
| `KvmCpuid::update_runtime` post: cpuid_dynamic_bits_dirty == false | `KvmCpuid::update_runtime` |
| `KvmCpuid::after_set_cpuid` post: cpu_caps[i] == (KVM_CPU_CAPS[i] | emulated[i]) & guest[i] | `KvmCpuid::after_set_cpuid` |
| `KvmCpuid::out_of_range_entry` post: AMD/Hygon ⟹ returns None | `KvmCpuid::out_of_range_entry` |
| `xstate_required_size` post: returns ≥ XSAVE_HDR_SIZE + XSAVE_HDR_OFFSET | `KvmCpuid::xstate_required_size` |

### Layer 4: Verus/Creusot functional

`per-CPUID-ioctl-install → per-runtime-update → per-emulate → per-GPR-writeback` semantic equivalence: per-Documentation/virt/kvm/api.rst §`KVM_SET_CPUID2`, §`KVM_GET_CPUID2`, §`KVM_GET_SUPPORTED_CPUID`, §`KVM_GET_EMULATED_CPUID`, and per-Documentation/virt/kvm/x86/cpuid.rst (`KVM_CPUID_FEATURES` leaf semantics and per-feature flag definitions).

## Hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

CPUID-emulation reinforcement:

- **Per-`KVM_MAX_CPUID_ENTRIES` cap (256)** — defense against per-userspace-DoS via huge CPUID arrays.
- **Per-`vmemdup_array_user` copy-once** — defense against per-TOCTOU-on-CPUID-blob.
- **Per-padding-must-be-zero strict check (`sanity_check_entries`)** — defense against per-UAPI-padding-smuggling of future fields.
- **Per-`kvm_cpu_caps` host-feature mask intersection** — defense against per-userspace-advertising-unsupported-feature.
- **Per-`kvm_check_cpuid` vaddr_bits ∈ {0, 48, 57} validation** — defense against per-invalid-canonical-check-assumption.
- **Per-`fpu_enable_guest_xfd_features` gate for dynamic XSAVE** — defense against per-FPU-state-without-host-XFD-enable.
- **Per-`kvm_cpuid_check_equal` post-RUN immutability** — defense against per-mid-execution-MAXPHYADDR-change (MMU desync, L2-state corruption).
- **Per-`lockdep_assert_irqs_enabled` in lookup** — defense against per-hot-path-CPUID-in-IRQ-disabled (performance regression).
- **Per-`get_cpu()/put_cpu()` around `do_host_cpuid`** — defense against per-cross-CPU-CPUID-inconsistency.
- **Per-PV-features quirk (`kvm_hlt_in_guest` strips PV_UNHALT)** — defense against per-conflicting-PV-feature-advertisement.
- **Per-AMD/Hygon no max-basic-redirect** — defense against per-vendor-spec-violation.
- **Per-`kvm_can_set_cpuid_and_feature_msrs` gate** — defense against per-post-KVM_RUN-CPUID-replacement.
- **Per-`KVM_REQ_RECALC_INTERCEPTS` request on set** — defense against per-stale-vendor-intercepts (VMX/SVM CPU-feature emulation).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `arch/x86/kvm/x86.c` core vCPU ioctl dispatch (covered separately if expanded; this doc only depicts the CPUID ioctls).
- `arch/x86/kvm/hyperv.c` Hyper-V emulation proper (this doc only references `kvm_hv_set_cpuid` and `kvm_cpuid_has_hyperv`).
- `arch/x86/kvm/xen.c` Xen emulation proper (this doc only references the Xen TSC leaf and `XEN_SIGNATURE` lookup).
- `arch/x86/kvm/pmu.c` PMU emulation proper (this doc only references the CPUID.0xA / 0x80000022 synthesis from `kvm_pmu_cap`).
- `arch/x86/kvm/mmu/` MMU integration (this doc only references `kvm_mmu_after_set_cpuid` and MAXPHYADDR/reserved-GPA-bits handoff).
- VMX / SVM `vcpu_after_set_cpuid` vendor callbacks (covered in `kvm.md` and vendor-specific Tier-3 docs).
- KVM_CAP capability enumeration ioctls (separate from CPUID).
- Implementation code.
