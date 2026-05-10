# Tier-3: arch/x86/kvm/x86.c (MCE subset) — KVM MCA (Machine-Check Architecture) virtualization (per-vCPU MCG/MCx banks + KVM_X86_SETUP_MCE + inject_mce)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/x86.c (MCE-related sections; see grep "MCG\|MCE\|kvm_mce")
  - arch/x86/include/uapi/asm/msr-index.h (MSR_IA32_MCG_*, MSR_IA32_MCx_*)
  - include/uapi/linux/kvm.h (KVM_X86_SETUP_MCE + KVM_X86_GET_MCE_CAP_SUPPORTED)
-->

## Summary

Intel/AMD MCA (Machine-Check Architecture) is the CPU mechanism for hardware-fault reporting via per-bank Machine-Check Banks (MCx) — each bank reports per-component fault info (memory ECC, cache parity, bus error, etc.). KVM virtualizes MCA so guests can configure their own MC banks, receive injected MCEs (e.g., from userspace fault-injection or host-MCE that targets guest memory), and process #MC exceptions. Per-VM `mcg_cap` defines bank count + features; per-vCPU shadow of MSR_IA32_MCG_CTL/_STATUS + per-bank MSR_IA32_MCx_CTL/_STATUS/_ADDR/_MISC. KVM_X86_SETUP_MCE ioctl configures per-vCPU; KVM_X86_SET_MCE injects to guest.

This Tier-3 covers MCE-related code in `arch/x86/kvm/x86.c` (~200-300 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_arch_vcpu_ioctl_setup_mce(vcpu, mcg_cap)` | KVM_X86_SETUP_MCE | `Vcpu::ioctl_setup_mce` |
| `kvm_vcpu_ioctl_x86_set_mce(vcpu, &mce)` | KVM_X86_SET_MCE | `Vcpu::ioctl_set_mce` |
| `kvm_set_mce_*` MSR handlers (in MSR dispatch) | guest WRMSR(MCx) | covered in `Vcpu::handle_msr` |
| `kvm_get_mce_*` MSR handlers | guest RDMSR(MCx) | covered in `Vcpu::handle_msr` |
| `MSR_IA32_MCG_CAP` (0x179) | per-vCPU bank cap | UAPI |
| `MSR_IA32_MCG_STATUS` (0x17A) | per-vCPU global status | UAPI |
| `MSR_IA32_MCG_CTL` (0x17B) | per-vCPU global ctrl | UAPI |
| `MSR_IA32_MCG_EXT_CTL` (0x4D0) | per-vCPU extended ctrl (LMCE) | UAPI |
| `MSR_IA32_MC0_CTL`..`MCx_CTL(N-1)` | per-bank ctrl | UAPI |
| `MSR_IA32_MC0_STATUS`..`MCx_STATUS(N-1)` | per-bank status | UAPI |
| `MSR_IA32_MC0_ADDR`..`MCx_ADDR(N-1)` | per-bank addr | UAPI |
| `MSR_IA32_MC0_MISC`..`MCx_MISC(N-1)` | per-bank misc | UAPI |
| `MSR_IA32_MC0_CTL2`..`MCx_CTL2(N-1)` | per-bank CMCI threshold | UAPI |
| `KVM_MAX_MCE_BANKS` | per-vCPU bank count cap | UAPI |
| `KVM_X86_GET_MCE_CAP_SUPPORTED` | KVM-supported caps | UAPI |
| `kvm_x86_inject_exception(vcpu, MC_VECTOR, ...)` | inject #MC | shared |

## Compatibility contract

REQ-1: Per-vCPU MCA state in `vcpu.arch`:
- `mcg_cap` (MSR_IA32_MCG_CAP shadow):
  - Bits[7:0]: number of banks (1-32).
  - Bit[8]: MCG_CTL_P (MSR_IA32_MCG_CTL present).
  - Bit[9]: MCG_EXT_P (extended-ctl present).
  - Bit[10]: MCG_CMCI_P (corrected machine-check interrupt present).
  - Bit[11]: MCG_TES_P (threshold-exceeded support).
  - Bit[24]: MCG_LMCE_P (local MCE).
- `mcg_status` (MSR_IA32_MCG_STATUS).
- `mcg_ctl` (MSR_IA32_MCG_CTL; per-bank-en bitmap).
- `mcg_ext_ctl` (MSR_IA32_MCG_EXT_CTL; LMCE-en).
- `mci[BANKS]` (per-bank state):
  - ctl, status, addr, misc, ctl2.

REQ-2: KVM_X86_SETUP_MCE ioctl:
- userspace passes mcg_cap (number of banks + features).
- KVM validates: bank count ≤ KVM_MAX_MCE_BANKS; features ⊆ host-supported.
- vcpu.arch.mcg_cap = mcg_cap.
- Allocate vcpu.arch.mci[banks].

REQ-3: KVM_X86_SET_MCE ioctl (inject MCE to guest):
- userspace passes `kvm_x86_mce`: { status, mcg_status, addr, misc, bank }.
- KVM:
  - vcpu.arch.mci[bank].status = status.
  - vcpu.arch.mci[bank].addr = addr.
  - vcpu.arch.mci[bank].misc = misc.
  - vcpu.arch.mcg_status |= MCG_STATUS_MCIP (MCE in progress).
  - kvm_x86_inject_exception(vcpu, MC_VECTOR).
- Guest #MC handler reads MCx MSRs.

REQ-4: Per-MSR write (WRMSR handler):
- MSR_IA32_MCG_STATUS: shadow update.
- MSR_IA32_MCG_CTL: shadow update; per-bank enable mask change.
- MSR_IA32_MCG_EXT_CTL: shadow update.
- MSR_IA32_MCx_CTL: per-bank ctl update.
- MSR_IA32_MCx_STATUS: typically write-1-to-clear; clears bank status.
- MSR_IA32_MCx_ADDR / _MISC: per-bank addr/misc update.
- MSR_IA32_MCx_CTL2: per-bank CMCI threshold.

REQ-5: Per-MSR read (RDMSR handler):
- Return per-vCPU shadow value.

REQ-6: Per-bank validation:
- Bank index < (mcg_cap & MCG_CAP_BANKCNT_MASK).
- Out-of-range MSRs return -EINVAL or #GP.

REQ-7: Reserved-bits:
- Per-MSR reserved-bits ignored on write per Intel SDM.
- KVM masks per-MSR_RESERVED_MASK per-spec.

REQ-8: KVM_X86_GET_MCE_CAP_SUPPORTED ioctl:
- Returns host-MCE features that KVM can virtualize.
- Userspace queries before SETUP_MCE.

REQ-9: Inject-MCE flow:
1. KVM_X86_SET_MCE ioctl with bank-info.
2. vcpu.arch.mcg_status updated.
3. vcpu.arch.mci[bank] populated.
4. kvm_x86_inject_exception(vcpu, MC_VECTOR, ...).
5. Per-vmenter: VM_ENTRY_INTR_INFO_FIELD with #MC vector.
6. Guest #MC handler runs; reads MCx MSRs.

REQ-10: Live migration:
- All MCA MSRs included in KVM_GET_MSRS migration list.
- Per-vCPU mcg_cap migrated.

REQ-11: LMCE (Local MCE):
- LMCE = local MC; per-vCPU rather than broadcast.
- Gated by MCG_EXT_CTL.LMCE_EN.
- KVM advertises LMCE support if host CPU supports.

REQ-12: CMCI (Corrected Machine-Check Interrupt):
- Per-bank CMCI when corrected-error count exceeds threshold.
- KVM emulates via per-bank ctl2 + LVT_CMCI (LAPIC virtualization).

## Acceptance Criteria

- [ ] AC-1: KVM_X86_GET_MCE_CAP_SUPPORTED returns host-supported caps; userspace queries successfully.
- [ ] AC-2: KVM_X86_SETUP_MCE: per-vCPU configure with 6 banks + LMCE + CMCI; subsequent guest CPUID shows MCE.
- [ ] AC-3: Boot Linux guest with `/proc/cpuinfo`: mce + cmci flags visible.
- [ ] AC-4: Guest reads MSR_IA32_MCG_CAP: returns configured value.
- [ ] AC-5: Inject MCE: KVM_X86_SET_MCE with bank=2 + status=uncorrectable; guest #MC handler runs.
- [ ] AC-6: Guest writes MCx_STATUS=0: clears bank status; subsequent read returns 0.
- [ ] AC-7: LMCE: enable via MCG_EXT_CTL.LMCE_EN; subsequent #MC delivered to specific vCPU rather than broadcast.
- [ ] AC-8: CMCI: per-bank threshold exceeded; LVT_CMCI fires on guest LAPIC.
- [ ] AC-9: Live migration: per-vCPU MCA MSRs preserved; post-migrate guest reads consistent.
- [ ] AC-10: kvm-unit-tests `mce` test passes.

## Architecture

Per-vCPU MCA state extension (in vcpu.arch):

```
struct VcpuArchMca {
  mcg_cap: u64,                                 // MSR_IA32_MCG_CAP
  mcg_status: u64,                               // MSR_IA32_MCG_STATUS
  mcg_ctl: u64,                                  // MSR_IA32_MCG_CTL
  mcg_ext_ctl: u64,                              // MSR_IA32_MCG_EXT_CTL
  mci: KVec<KvmMceBank>,                         // per-bank
}

struct KvmMceBank {
  ctl: u64,                                      // MSR_IA32_MCx_CTL
  status: u64,                                   // MSR_IA32_MCx_STATUS
  addr: u64,                                     // MSR_IA32_MCx_ADDR
  misc: u64,                                     // MSR_IA32_MCx_MISC
  ctl2: u64,                                     // MSR_IA32_MCx_CTL2
}
```

`Vcpu::ioctl_setup_mce(vcpu, mcg_cap)`:
1. banks := mcg_cap & 0xFF.
2. If banks > KVM_MAX_MCE_BANKS: return -EINVAL.
3. Validate features bits ⊆ host-supported.
4. vcpu.arch.mca.mcg_cap = mcg_cap.
5. Allocate vcpu.arch.mca.mci[banks]; init to zero.

`Vcpu::ioctl_set_mce(vcpu, &mce)`:
1. bank := mce.bank.
2. If bank >= banks: return -EINVAL.
3. vcpu.arch.mca.mci[bank].status = mce.status.
4. vcpu.arch.mca.mci[bank].addr = mce.addr.
5. vcpu.arch.mca.mci[bank].misc = mce.misc.
6. vcpu.arch.mca.mcg_status |= MCG_STATUS_MCIP | (mce.mcg_status & STATUS_MASK).
7. kvm_x86_inject_exception(vcpu, MC_VECTOR, ...).
8. kvm_make_request(KVM_REQ_EVENT, vcpu).

`Vcpu::handle_msr` MCA branches:
- MSR_IA32_MCG_CAP: read-only; -EINVAL on write.
- MSR_IA32_MCG_STATUS / MCG_CTL / MCG_EXT_CTL: shadow R/W.
- MSR_IA32_MCx_CTL: per-bank ctl R/W.
- MSR_IA32_MCx_STATUS: write-1-to-clear (per-bit clear).
- MSR_IA32_MCx_ADDR / _MISC: per-bank R/W.
- MSR_IA32_MCx_CTL2: per-bank CMCI threshold R/W.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bank_idx_bounded` | OOB | per-bank idx < (mcg_cap & 0xFF); defense against OOB mci array. |
| `mcg_cap_features_subset` | INVARIANT | per-vCPU mcg_cap features ⊆ host-supported. |
| `inject_mce_atomic` | INVARIANT | inject_mce sets status + injects exception in one critical section. |
| `mcip_only_during_inject` | INVARIANT | MCG_STATUS_MCIP set during in-progress inject; cleared on completion. |
| `lmce_gated_on_ext_ctl` | INVARIANT | LMCE delivery gated by MCG_EXT_CTL.LMCE_EN. |

### Layer 2: TLA+

`virt/kvm/mce_inject.tla`:
- Per-vCPU MCA state: per-bank (clean / valid).
- Per-MCE-inject lifecycle: pending, in-progress, delivered.
- Properties:
  - `safety_one_mce_at_a_time` — per-vCPU at most one MCIP at a time.
  - `safety_status_persists_until_clear` — bank status persists until guest WRMSR(MCx_STATUS, 0).
  - `liveness_inject_eventually_delivered` — KVM_X86_SET_MCE eventually delivers #MC.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::ioctl_setup_mce` post: vcpu.arch.mca.mcg_cap valid; mci[] allocated | `Vcpu::ioctl_setup_mce` |
| `Vcpu::ioctl_set_mce` post: bank populated; #MC inject queued | `Vcpu::ioctl_set_mce` |
| `Vcpu::handle_msr` MCA branch: per-MSR shadow R/W consistent | `Vcpu::handle_msr` |
| Per-vCPU mci[bank].status WRMSR write-1-to-clear semantics | per-MSR handler |

### Layer 4: Verus/Creusot functional

`KVM_X86_SET_MCE → guest #MC handler reads MSR_IA32_MCx_STATUS = expected status` semantic equivalence: per-inject the bank fields visible to guest match userspace-supplied values.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

MCA-specific reinforcement:

- **Per-vCPU bank count ≤ KVM_MAX_MCE_BANKS** — defense against unbounded mci array.
- **mcg_cap features ⊆ host-supported** — defense against guest using unsupported features causing host CPU exception.
- **Per-MSR reserved-bits masked** — defense against guest writing reserved bits causing CPU exception.
- **Per-bank.status write-1-to-clear** — defense against bit-flip attacks on status.
- **MCG_STATUS_MCIP transitions** — defense against torn state during inject.
- **LMCE gated** — defense against per-vCPU MCE leaking to other vCPUs unintentionally.
- **CMCI gated** — defense against guest CMCI without LVT_CMCI being set.
- **Live-migrate per-vCPU MCA preserved** — defense against post-migrate state inconsistency.
- **KVM_X86_GET_MCE_CAP_SUPPORTED** — userspace can query before configure to avoid unsupported features.
- **Per-vCPU MCA emulation isolated** — defense against host-MCE spilling into other VMs' guests.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- LAPIC LVT_CMCI delivery (covered in `x86-lapic.md` Tier-3)
- Host-side MCE driver (drivers/cpu/x86/mce; covered in `arch/x86/mce.md` future Tier-3)
- Implementation code
