# Tier-3: arch/x86/kvm/cpuid.c — CPUID virtualization (per-vCPU feature mask + KVM_GET_SUPPORTED_CPUID + dynamic CPUID rewrite)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/00-overview.md
upstream-paths:
  - arch/x86/kvm/cpuid.c
  - arch/x86/kvm/cpuid.h
  - arch/x86/kvm/reverse_cpuid.h
  - include/uapi/linux/kvm.h (KVM_GET_SUPPORTED_CPUID + KVM_GET_EMULATED_CPUID + KVM_SET_CPUID + KVM_SET_CPUID2 + KVM_GET_CPUID2)
-->

## Summary

CPUID virtualization is what every KVM guest sees when it executes the `CPUID` instruction — KVM reports per-VM-configured + per-vCPU-configured CPUID values rather than raw host CPUID. Userspace VMM (qemu, cloud-hypervisor) calls `KVM_GET_SUPPORTED_CPUID` ioctl to enumerate KVM-supported guest CPUID leaves, then masks per-VM-policy + calls `KVM_SET_CPUID2` to install per-vCPU CPUID table. On every guest CPUID instruction execution, KVM consults this per-vCPU table and returns matching leaf values; some KVM-specific leaves (KVM_CPUID_FEATURES at 0x40000001, KVM_CPUID_SIGNATURE at 0x40000000) advertise paravirt features.

Critical for: SMP guest topology presentation, feature-flag advertising (which CPUID feature bits guest can rely on — e.g., AVX512 disabled if migrating across mixed CPU generations), per-vCPU APIC ID assignment, paravirt feature negotiation (kvm-clock, async-pf, pv-tlb-flush, pv-spinlock, etc.).

This Tier-3 covers `arch/x86/kvm/cpuid.c` (~2200 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_cpuid_entry2` | per-leaf CPUID entry (UAPI) | `uapi::KvmCpuidEntry2` |
| `kvm_check_cpuid(vcpu, &entries, nent)` | validate proposed CPUID set | `Vcpu::check_cpuid` |
| `kvm_set_cpuid(vcpu, &entries, nent)` | install per-vCPU CPUID table | `Vcpu::set_cpuid` |
| `kvm_vcpu_after_set_cpuid(vcpu)` | post-set update of derived per-vCPU state | `Vcpu::after_set_cpuid` |
| `kvm_dev_ioctl_get_supported_cpuid(cpuid, entries)` | KVM_GET_SUPPORTED_CPUID ioctl | `DevKvm::get_supported_cpuid` |
| `kvm_dev_ioctl_get_emulated_cpuid(cpuid, entries)` | KVM_GET_EMULATED_CPUID ioctl | `DevKvm::get_emulated_cpuid` |
| `kvm_vcpu_ioctl_set_cpuid2(vcpu, cpuid, entries)` | KVM_SET_CPUID2 ioctl | `Vcpu::ioctl_set_cpuid2` |
| `kvm_vcpu_ioctl_get_cpuid2(vcpu, cpuid, entries)` | KVM_GET_CPUID2 ioctl | `Vcpu::ioctl_get_cpuid2` |
| `kvm_emulate_cpuid(vcpu)` | runtime CPUID instruction emulation | `Vcpu::emulate_cpuid` |
| `kvm_find_cpuid_entry(vcpu, function, index)` / `_index` | per-leaf lookup | `Vcpu::find_cpuid_entry` |
| `kvm_cpu_cap_init(...)` / `_init_kvm_defined_features` | KVM-side cpu-cap mask init | `Subsystem::cpu_cap_init` |
| `kvm_cpu_cap_check_and_set(leaf, feature)` | per-feature cap-set | `CpuCap::check_and_set` |
| `kvm_cpu_cap_set_emulated(leaf, feature)` | mark feature as KVM-emulated | `CpuCap::set_emulated` |
| `cpuid_query_maxphyaddr(vcpu)` | per-vCPU max-phys-addr CPUID-derived | `Vcpu::maxphyaddr` |
| `kvm_set_msr_filter(...)` | per-vCPU MSR access filter (CPUID-related) | `Vcpu::set_msr_filter` |
| `kvm_apply_cpuid_pv_features_quirk(vcpu)` | quirk for paravirt feature consistency | `Vcpu::apply_pv_features_quirk` |
| `do_cpuid_func(vcpu, function, &entry, type)` / `do_host_cpuid(...)` | dynamic CPUID populate per-leaf | `Vcpu::do_cpuid_func` / `_do_host_cpuid` |

## Compatibility contract

REQ-1: `KVM_GET_SUPPORTED_CPUID` UAPI byte-identical:
- Returns array of `kvm_cpuid_entry2` for all KVM-supported leaves.
- Per-leaf: function (0x0..0xN std + 0x40000000+ KVM-PV + 0x80000000+ extended + 0x80860000+ Centaur).
- Per-leaf eax/ebx/ecx/edx fields.
- Some leaves have sub-functions (indexed via `KVM_CPUID_FLAG_SIGNIFCANT_INDEX` flag).

REQ-2: `KVM_GET_EMULATED_CPUID`: similar but only KVM-emulated features (subset where KVM emulates HW instruction not natively present).

REQ-3: `KVM_SET_CPUID2(vcpu, cpuid, entries)`:
- Validate entries: `nent <= KVM_MAX_CPUID_ENTRIES (256)`.
- Per-leaf validation: must be in supported set (or hyperv/KVM-PV synthetic).
- Cross-leaf consistency: e.g., CPUID.0x07.0:EBX.AVX512F bit only set if CPUID.0x01:ECX.OSXSAVE set.
- Post-set: `Vcpu::after_set_cpuid` updates derived state (per-vCPU max-phys-addr, MSR-bitmap, hyperv-feature-mask, kvm-pv-feature-mask, AVX/SSE state-save bits).

REQ-4: Per-vCPU CPUID table indexed by function + index for sub-leaf lookups; per-leaf flags (SIGNIFCANT_INDEX / STATEFUL_FUNC).

REQ-5: KVM-PV CPUID leaves at 0x40000000-0x4000FFFF:
- 0x40000000: SIGNATURE (string "KVMKVMKVM\0\0\0").
- 0x40000001: FEATURES (KVM_FEATURE_CLOCKSOURCE / _NOP_IO_DELAY / _MMU_OP / _CLOCKSOURCE2 / _ASYNC_PF / _STEAL_TIME / _PV_EOI / _PV_UNHALT / _PV_TLB_FLUSH / _ASYNC_PF_VMEXIT / _PV_SEND_IPI / _POLL_CONTROL / _PV_SCHED_YIELD / _ASYNC_PF_INT / _MSI_EXT_DEST_ID / _HC_MAP_GPA_RANGE / _MIGRATION_CONTROL).

REQ-6: HyperV CPUID leaves at 0x40000005-0x4000000F (when KVM_CAP_HYPERV enabled): emulate Hyper-V hypercall + msrs for Windows guest compat.

REQ-7: Xen CPUID leaves at 0x40000200+ (when KVM_CAP_XEN_HVM enabled): emulate Xen hypercall surface for Xen guests.

REQ-8: Per-vCPU CPUID instruction emulation (`Vcpu::emulate_cpuid`):
- Read EAX (function) + ECX (sub-function).
- `kvm_find_cpuid_entry(vcpu, function, index)` → returns matching entry.
- Set guest registers eax/ebx/ecx/edx from entry.
- Some leaves dynamically computed (e.g., CPUID.0x0F sub-RMID enumeration; CPUID.0x14 Intel-PT enumeration).

REQ-9: Reverse CPUID mapping (`reverse_cpuid.h`): per-feature bit-position → CPUID-leaf encoding; used to extract supported features from `cpu_caps[]` arrays and translate to CPUID format.

REQ-10: Per-vCPU max-phys-addr (`maxphyaddr`): derived from CPUID.0x80000008:EAX[7:0]; used by paging-walk + EPT/NPT setup; per-spec must match guest expectation across vCPUs in same VM.

REQ-11: Per-vCPU APIC ID (CPUID.0x01:EBX[31:24]): SMP topology config; vCPU index → x2APIC-ID mapping.

REQ-12: KVM_SET_CPUID2 + post-set: triggers MSR-bitmap recompute (allow guest direct access to MSRs supported via CPUID flags, deny otherwise), XSAVE state-save bitmap recompute, hyperv-msr enable/disable.

## Acceptance Criteria

- [ ] AC-1: qemu Linux guest test: `cat /proc/cpuinfo` shows guest-visible features matching qemu's `-cpu host` config; AVX/AVX2/AVX512 / SSE / VMX advertised correctly.
- [ ] AC-2: SMP topology: 4-vCPU guest sees 4 CPUs via CPUID.0x01:EBX[23:16] (logical-cpu-count).
- [ ] AC-3: KVM-PV features test: `dmesg | grep "kvm-clock"` in guest shows kvm-pvclock advertised + initialized; CPUID.0x40000001:EAX KVM_FEATURE_CLOCKSOURCE2 bit set.
- [ ] AC-4: Hyper-V emulation: `qemu -cpu Hyper-V-passthrough` Windows guest sees Hyper-V CPUID leaves + boots without BSOD.
- [ ] AC-5: KVM_GET_SUPPORTED_CPUID returns expected per-host CPUID set; qemu uses to mask per-VM features.
- [ ] AC-6: Live migration: KVM_GET_CPUID2 round-trip preserves per-vCPU CPUID exactly; migrated guest sees same CPUID values.
- [ ] AC-7: Cross-leaf consistency: KVM_SET_CPUID2 with AVX512 set but OSXSAVE clear rejected with -EINVAL.
- [ ] AC-8: kvm-unit-tests `cpuid` test passes.

## Architecture

`Vcpu` extension `kernel::kvm::x86::Vcpu` includes:

```
struct Vcpu {
  ...
  cpuid_entries: KBox<[KvmCpuidEntry2; KVM_MAX_CPUID_ENTRIES]>,  // 256 entries
  cpuid_nent: u32,
  // derived state from CPUID:
  arch_capabilities: u64,                 // CPUID-derived; mirrored in MSR_IA32_ARCH_CAPABILITIES emulation
  pv_features: u32,                       // KVM-PV feature bitmap
  hyperv_caps: KBox<KvmHypervFeatures>,
  max_phyaddr: u8,
  ...
}
```

Subsystem-wide `cpu_caps[]` array (per-leaf bitmap of KVM-supported features):

```
static cpu_caps: PerCpuCaps;                 // global; per-leaf u32 of supported feature bits
```

`Subsystem::cpu_cap_init`:
1. For each known CPUID leaf: read host CPUID; intersect with KVM's policy (some bits force-cleared if KVM doesn't support virtualization, some force-set for KVM-PV).
2. Populate `cpu_caps[leaf] = host_value & KVM_SUPPORTED_MASK`.
3. Mark KVM-emulated features via `kvm_cpu_cap_set_emulated(leaf, feature)`.

`DevKvm::get_supported_cpuid(cpuid_arg)`:
1. For each leaf in supported list:
   - `do_host_cpuid(...)` populates entry from current host CPUID + per-leaf KVM mask.
   - Special handling for KVM-PV leaves (synthetic).
   - Special handling for Hyper-V leaves (when CONFIG_KVM_HYPERV=y).
2. Copy entry array to userspace.

`Vcpu::set_cpuid(vcpu, &entries, nent)`:
1. Validate `nent <= KVM_MAX_CPUID_ENTRIES`.
2. `Vcpu::check_cpuid(vcpu, &entries, nent)`:
   - Per-entry: function in valid range; flags valid.
   - Cross-entry consistency checks:
     - AVX512 requires OSXSAVE.
     - x2APIC requires xAPIC-not-disabled.
     - MTRR requires PSE.
   - Per-vendor validation for Intel/AMD-specific leaves.
3. Copy entries into vcpu.cpuid_entries.
4. `Vcpu::after_set_cpuid(vcpu)`:
   - Recompute MSR bitmap (allow direct access to architectural MSRs based on CPUID-advertised features).
   - Recompute XSAVE state-save bitmap (XCR0 vs CPUID.0xD).
   - Update vcpu.maxphyaddr from CPUID.0x80000008.
   - Update vcpu.pv_features from CPUID.0x40000001.
   - Update hyperv-related state if hyperv-enabled.
   - Update guest-visible CR0/CR4 reserved bits per CPUID.

`Vcpu::emulate_cpuid(vcpu)`:
1. Read function from EAX, sub-function from ECX.
2. `Vcpu::find_cpuid_entry(vcpu, function, index)`:
   - Walk `vcpu.cpuid_entries`; find matching function (+ index if SIGNIFCANT_INDEX flag).
   - If not found: return entry with all-zero (per Intel SDM behavior for unsupported leaves).
3. Set guest EAX = entry.eax; EBX = entry.ebx; ECX = entry.ecx; EDX = entry.edx.
4. Some leaves dynamically computed:
   - CPUID.0x0B/0x1F (extended topology): per-vCPU APIC ID derived from vcpu_idx.
   - CPUID.0xD sub-leaves (XSAVE state component): per-feature size derived from XCR0 capability.
   - CPUID.0x14 (Intel-PT): per-vCPU PT capability.
5. Advance RIP past CPUID instruction.

KVM-PV signature/features (`Vcpu::do_cpuid_func` for 0x40000000+):
- 0x40000000: EAX = highest-supported-PV-leaf (0x40000010+), EBX/ECX/EDX = "KVMKVMKVM\0\0\0" string in 12-byte ASCII.
- 0x40000001: EAX = KVM_FEATURE_* bitmap; EDX = additional flags (e.g., HLT_HC_VAPIC_POLL_IRQ).
- 0x40000010: EAX = TSC frequency (kHz; kvm-clock related).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cpuid_entries_no_oob` | OOB | per-vCPU cpuid_entries array indexed by entry-iter; bounded by nent ≤ KVM_MAX_CPUID_ENTRIES. |
| `cross_leaf_consistent` | INVARIANT | KVM_SET_CPUID2 validation enforces cross-leaf dependencies (AVX512 ↔ OSXSAVE, x2APIC ↔ xAPIC, etc.). |
| `pv_signature_correct` | CONSTANT | KVM-PV CPUID 0x40000000 signature always "KVMKVMKVM\0\0\0"; defense against signature divergence breaking guest paravirt detection. |
| `maxphyaddr_consistent` | INVARIANT | per-vCPU max_phyaddr matches CPUID.0x80000008:EAX[7:0]; per-paging-walk + EPT setup uses this. |

### Layer 2: TLA+

(No specific TLA+ — semantics covered by per-instruction emulation correctness in `x86-emulate.md` Tier-3.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::set_cpuid` post: vcpu.cpuid_entries populated; vcpu.cpuid_nent updated; derived state recomputed | `Vcpu::set_cpuid` |
| `Vcpu::find_cpuid_entry` post: returned entry (if Some) has `function == requested && (no SIGNIFCANT_INDEX OR index == requested)` | `Vcpu::find_cpuid_entry` |
| Per-vCPU MSR-bitmap consistent with CPUID-advertised features after KVM_SET_CPUID2 | `Vcpu::after_set_cpuid` |
| `KVM_GET_CPUID2` returns exact set just installed via `KVM_SET_CPUID2` | `Vcpu::ioctl_get_cpuid2` |

### Layer 4: Verus/Creusot functional

`KVM_SET_CPUID2(vcpu, entries) → guest CPUID instruction → emulated entry returned to guest registers` round-trip equivalence: per-leaf function value seen by guest matches userspace-supplied entry values + per-vCPU APIC ID + per-vCPU max_phyaddr derivation.

## Hardening

(Inherits row-1 features from `virt/kvm/00-overview.md` § Hardening.)

cpuid-specific reinforcement:

- **Per-vCPU cpuid_entries cap** (KVM_MAX_CPUID_ENTRIES = 256) — defense against per-VM CPUID-flood causing memory exhaustion.
- **Cross-leaf consistency validation** at KVM_SET_CPUID2 — defense against guest exploiting inconsistent CPUID (e.g., AVX512-set + OSXSAVE-clear → guest #UD bypass attack).
- **KVM-PV signature constant** — defense against userspace-controllable signature confusing guest paravirt detection.
- **Per-feature MSR-bitmap derivation** — guest cannot RDMSR/WRMSR for MSRs not advertised via CPUID + supported by KVM; defense against feature-bypass attack.
- **Per-vCPU max_phyaddr capped at host max_phyaddr** — defense against guest claiming wider addr-space than HW supports.
- **CPUID.0xD sub-leaf XSAVE size validated against host XCR0** — defense against guest XSAVE buffer-size mismatch causing #PF or #GP.
- **Hyper-V CPUID leaves gated by KVM_CAP_HYPERV** — defense against accidentally enabling Hyper-V emulation; per-VM opt-in.
- **Xen CPUID leaves gated by KVM_CAP_XEN_HVM** — same, per-VM opt-in for Xen guest support.
- **Per-vCPU APIC ID validated** — must be ≤ KVM_MAX_VCPU_IDS (4096); defense against APIC ID overflow.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — KVM_SET_CPUID2 array bounded by KVM_MAX_CPUID_ENTRIES * sizeof(entry).
- **PAX_KERNEXEC** — cpuid emulation dispatch table RO after init.
- **PAX_RANDKSTACK** — randomized kstack per KVM_SET_CPUID2.
- **PAX_REFCOUNT** — cpuid_entries refcount on per-vCPU array saturating.
- **PAX_MEMORY_SANITIZE** — cpuid_entries freed via kvfree → zeroed.
- **PAX_UDEREF** — user CPUID array pointer validated before copy_from_user.
- **PAX_RAP / kCFI** — kvm_cpuid / kvm_emulate_cpuid call sites type-checked.
- **GRKERNSEC_HIDESYM** — cpuid_entries pointer redacted in error logs.
- **GRKERNSEC_DMESG** — repeated CPUID set failures rate-limited.

CPUID-specific:

- **CAP_SYS_ADMIN strict on KVM_SET_CPUID2** — defense against unprivileged CPUID flip.
- **Cross-leaf consistency check inside locked critical section** — TOCTOU-safe.
- **Hyper-V / Xen leaves gated by per-VM cap + capable()** — defense against accidental paravirt-vendor switch.

Rationale: CPUID is the policy oracle for every MSR/feature gate; sanitize/refcount on the entry array + RAP-checked dispatch ensure a malicious userspace cannot smuggle a stale-pointer post-set to bypass MSR-filter or feature-MSR locks.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- Per-instruction emulation (covered in `x86-emulate.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- Hyper-V emulation details (covered in `virt/kvm/x86-hyperv.md` future Tier-3)
- Xen emulation details (covered in `virt/kvm/x86-xen.md` future Tier-3)
- KVM clock + paravirt (covered in `virt/kvm/x86-pvclock.md` future Tier-3)
- 32-bit-only paths
- Implementation code
