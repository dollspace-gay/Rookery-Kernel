# Tier-3: arch/x86/kvm/lapic.c (PV-EOI subset) — KVM paravirtual EOI optimization

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-lapic.md
upstream-paths:
  - arch/x86/kvm/lapic.c (PV-EOI: ~900..930, ~3331..3350, ~3503..3525)
  - arch/x86/kvm/x86.c (MSR_KVM_PV_EOI_EN: ~422, ~4184, ~4541)
  - arch/x86/include/uapi/asm/kvm_para.h (KVM_PV_EOI_BIT)
-->

## Summary

PV-EOI is a paravirtual optimization (KVM_FEATURE_PV_EOI) that reduces vmexits during interrupt delivery: KVM sets a per-vCPU shared-memory flag (`KVM_PV_EOI_ENABLED`) at IRQ-injection time; guest checks the flag in its EOI-handler — if enabled, guest skips the explicit APIC-EOI MMIO write (which would vmexit) and clears the flag in shared memory. KVM polls the flag at next IRQ-inject. Per-vCPU MSR `MSR_KVM_PV_EOI_EN` (0x4b564d04) configures the GPA of the flag-byte. Critical for: high-IRQ-rate guest workloads (e.g. 100Gbps NIC with virtio).

This Tier-3 covers PV-EOI subset of `lapic.c` (~80 lines) + `x86.c` MSR handler.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `MSR_KVM_PV_EOI_EN` (0x4b564d04) | per-vCPU PV-EOI flag GPA | UAPI |
| `KVM_PV_EOI_BIT` (bit 0) | per-vCPU PV-EOI enabled flag | UAPI |
| `KVM_PV_EOI_ENABLED` (1u8) | per-flag enabled value | UAPI |
| `KVM_PV_EOI_DISABLED` (0u8) | per-flag disabled value | UAPI |
| `KVM_APIC_PV_EOI_PENDING` (per-vCPU bit) | per-vCPU "EOI pending" marker | shared |
| `kvm_lapic_set_pv_eoi()` | per-vCPU enable | `Lapic::set_pv_eoi` |
| `pv_eoi_get_user()` / `pv_eoi_put_user()` | per-flag GPA read/write | `Lapic::pv_eoi_*` |
| `pv_eoi_get_pending()` | per-vCPU check pending-EOI | `Lapic::pv_eoi_get_pending` |
| `pv_eoi_set_pending()` | per-vCPU set | `Lapic::pv_eoi_set_pending` |
| `pv_eoi_clr_pending()` | per-vCPU clear | `Lapic::pv_eoi_clr_pending` |
| `kvm_pv_eoi_init()` | per-vCPU init | `Lapic::pv_eoi_init` |
| `__apic_accept_irq()` | per-vCPU IRQ-accept (sets flag) | `Lapic::__apic_accept_irq` |

## Compatibility contract

REQ-1: KVM_FEATURE_PV_EOI feature bit:
- Advertised via CPUID 0x40000001 EAX bit KVM_FEATURE_PV_EOI.
- Guest detects via cpuid_eax(KVM_CPUID_FEATURES) & (1 << KVM_FEATURE_PV_EOI).

REQ-2: MSR_KVM_PV_EOI_EN (0x4b564d04):
- Bits[0]: KVM_PV_EOI_ENABLED.
- Bits[63:0]: GPA of u8 flag-byte (8-byte aligned).
- Per-WRMSR: KVM stores GPA + KVM_PV_EOI_ENABLED bit; reads/writes flag at this GPA.
- Per-RDMSR: KVM returns last-written value.

REQ-3: Per-vCPU shared flag-byte at GPA:
- byte = 0: KVM_PV_EOI_DISABLED.
- byte = 1: KVM_PV_EOI_ENABLED (KVM has injected interrupt with PV-EOI, guest may skip MMIO EOI).

REQ-4: Per-IRQ-inject flow:
- KVM sets shared flag-byte = KVM_PV_EOI_ENABLED via pv_eoi_put_user.
- KVM marks vcpu.arch.apic.pending_pv_eoi (per-vCPU bit).
- Guest sees IRQ; runs IRQ handler; on iret-to-eoi: checks flag-byte.
  - If enabled: clears flag-byte; skips APIC-EOI MMIO.
  - Else: writes APIC-EOI MMIO (legacy path; vmexit).

REQ-5: Per-next-IRQ-inject KVM checks pending:
- pv_eoi_get_pending(vcpu) reads flag-byte.
- If still 1 (KVM_PV_EOI_ENABLED): KVM treats prev IRQ as un-EOI'd (no PV-EOI happened); processes accordingly.
- Else: KVM processes EOI for previous IRQ (clears LAPIC ISR-bit).

REQ-6: Per-vCPU init reset:
- vcpu.arch.pv_eoi.msr_val = 0 (disabled).
- vcpu.arch.apic.pending_pv_eoi = 0.

REQ-7: Per-LAPIC TPR-shadow-vmexit:
- When TPR < ISR-pending interrupt: KVM is responsible.
- Per-PV-EOI: works with TPR-shadow but skips MMIO EOI.

REQ-8: Per-EOI semantics:
- LAPIC ISR + IRR + TPR semantics preserved.
- Per-vCPU EOI clears highest-priority ISR-bit.

REQ-9: Per-vCPU live-migration:
- vcpu.arch.pv_eoi.msr_val migrated.
- Per-flag-byte at GPA stays in guest memory (persistent).
- Destination KVM resumes pv_eoi_get_pending state.

REQ-10: Per-MSR_KVM_PV_EOI_EN write only if !lapic_in_kernel:
- PV-EOI requires in-kernel LAPIC.
- If userspace-LAPIC: WRMSR returns -1 (#GP).

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest: dmesg "KVM: paravirt EOI present".
- [ ] AC-2: Guest WRMSR(MSR_KVM_PV_EOI_EN, gpa | 1): KVM stores msr_val.
- [ ] AC-3: Per-IRQ-inject: kvm_lapic_set_pv_eoi sets flag-byte = 1 in guest.
- [ ] AC-4: Guest IRQ-handler sees flag = 1: skips MMIO-EOI; clears flag.
- [ ] AC-5: Tracepoint kvm_pv_eoi(vector): per-IRQ logged.
- [ ] AC-6: Per-IRQ-rate measurement: vmexit count for EOI MMIO = 0 (replaced).
- [ ] AC-7: Disable: WRMSR(MSR_KVM_PV_EOI_EN, 0): subsequent IRQs use legacy EOI.
- [ ] AC-8: Per-VM userspace-LAPIC: WRMSR returns #GP.
- [ ] AC-9: Per-vCPU live-migration: PV-EOI state preserved.
- [ ] AC-10: kvm-unit-tests `pv_eoi` test passes.

## Architecture

Per-vCPU state:

```
struct VcpuArch {
  ...
  pv_eoi: PvEoiState,
  apic: KvmLapic,
}

struct PvEoiState {
  msr_val: u64,                                   // GPA | KVM_PV_EOI_ENABLED bit
}

struct KvmLapic {
  ...
  pending_pv_eoi: AtomicBool,                     // KVM-side bit
}
```

`Lapic::set_pv_eoi(vcpu, data, len) -> Result<()>`:
1. addr = data & ~KVM_PV_EOI_ENABLED.
2. If !(data & KVM_PV_EOI_ENABLED):
   - vcpu.arch.pv_eoi.msr_val = 0; return Ok.
3. If !lapic_in_kernel(vcpu): return Err(EINVAL).
4. Validate addr is u8-aligned.
5. vcpu.arch.pv_eoi.msr_val = data.
6. (Optional pre-validate addr accessible in guest mem).
7. Ok.

`Lapic::pv_eoi_get_user(vcpu, val) -> Result<()>`:
1. addr = vcpu.arch.pv_eoi.msr_val & ~KVM_PV_EOI_ENABLED.
2. read_guest_byte(vcpu, addr, val).

`Lapic::pv_eoi_put_user(vcpu, val) -> Result<()>`:
1. addr = vcpu.arch.pv_eoi.msr_val & ~KVM_PV_EOI_ENABLED.
2. write_guest_byte(vcpu, addr, val).

`Lapic::pv_eoi_set_pending(vcpu) -> Result<()>`:
1. If !(vcpu.arch.pv_eoi.msr_val & KVM_PV_EOI_ENABLED): return Ok.
2. pv_eoi_put_user(vcpu, KVM_PV_EOI_ENABLED)?
3. vcpu.arch.apic.pending_pv_eoi = true.

`Lapic::pv_eoi_get_pending(vcpu) -> bool`:
1. If !(vcpu.arch.pv_eoi.msr_val & KVM_PV_EOI_ENABLED): return false.
2. val = pv_eoi_get_user(vcpu)?
3. Return val & KVM_PV_EOI_ENABLED.

`Lapic::pv_eoi_clr_pending(vcpu)`:
1. vcpu.arch.apic.pending_pv_eoi = false.

`Lapic::__apic_accept_irq(apic, mode, vector, level, trig, dest_map)`:
1. Standard LAPIC-IRQ-accept logic.
2. After ISR-bit set: pv_eoi_set_pending(apic.vcpu) (best-effort).

`Lapic::process_pending_eoi(vcpu)`:
1. If !pv_eoi_get_pending(vcpu): return.
2. /* Guest already wrote 0; KVM should EOI the LAPIC for previous-vector. */
3. apic_set_eoi(vcpu.arch.apic).
4. pv_eoi_clr_pending(vcpu).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pv_eoi_addr_aligned` | INVARIANT | per-msr_val GPA u8-aligned. |
| `pv_eoi_disabled_iff_msr_zero` | INVARIANT | pv_eoi disabled ⟺ msr_val & ENABLED == 0. |
| `pending_implies_msr_enabled` | INVARIANT | apic.pending_pv_eoi ⟹ msr_val & ENABLED. |
| `set_pv_eoi_requires_in_kernel_lapic` | INVARIANT | per-set succeeds ⟹ lapic_in_kernel(vcpu). |

### Layer 2: TLA+

`virt/kvm/pv_eoi.tla`:
- Per-vCPU IRQ-inject + flag-set + guest-read + guest-clear + KVM-poll.
- Properties:
  - `safety_no_lost_eoi` — per-IRQ-inject: KVM-side ISR-bit eventually cleared.
  - `safety_no_double_eoi` — per-IRQ: ISR-bit cleared at most once.
  - `liveness_pending_eventually_processed` — per-pending-pv-eoi + guest-iret ⟹ KVM observes flag=0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Lapic::set_pv_eoi` post: msr_val updated; lapic_in_kernel verified | `Lapic::set_pv_eoi` |
| `Lapic::pv_eoi_set_pending` post: flag-byte == KVM_PV_EOI_ENABLED; pending_pv_eoi=true | `Lapic::pv_eoi_set_pending` |
| `Lapic::pv_eoi_get_pending` post: returned == flag-byte at GPA | `Lapic::pv_eoi_get_pending` |
| `Lapic::process_pending_eoi` post: ISR-bit cleared; pending_pv_eoi=false | `Lapic::process_pending_eoi` |

### Layer 4: Verus/Creusot functional

`Per-IRQ-inject KVM sets flag → guest checks flag in iret-handler → skips MMIO-EOI → KVM polls flag-cleared on next IRQ-inject` semantic equivalence: per-PV-EOI matches KVM paravirt EOI specification.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-lapic.md` § Hardening.)

PV-EOI-specific reinforcement:

- **Per-msr_val GPA aligned to u8** — defense against unaligned causing partial write.
- **Per-set requires in-kernel LAPIC** — defense against userspace-LAPIC corner case.
- **Per-flag-byte read/write under guest_memcpy** — defense against per-GPA invalid causing kernel oops.
- **Per-vCPU pv_eoi_set_pending best-effort** — defense against per-failure leaking IRQ injection.
- **Per-vCPU process_pending_eoi at next IRQ-inject** — defense against EOI-loss.
- **Per-VM live-migrate pv_eoi state** — defense against post-migrate guest writing wrong GPA.
- **Per-vCPU lapic_in_kernel check at WRMSR** — defense against MSR-set without infrastructure.
- **Per-disable WRMSR(0) cleanly disables** — defense against per-disable leaving stale state.
- **Per-flag-byte check before each subsequent IRQ-inject** — defense against per-IRQ stale flag.
- **Per-tracepoint kvm_pv_eoi rate-limited** — defense against per-IRQ trace-flood.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — PV-EOI flag-byte write/read uses `kvm_write_guest_cached`/`kvm_read_guest_offset_cached` against a bounded 1-byte gfn-cache; no user pointer ingress.
- **PAX_KERNEXEC** — `pv_eoi_set_pending`, `pv_eoi_get_pending`, `kvm_lapic_set_eoi_accelerated` resolve through RX-only kernel text.
- **PAX_RANDKSTACK** — IRQ-inject and WRMSR paths inherit RANDKSTACK from KVM_RUN / IRQ entry.
- **PAX_REFCOUNT** — per-vCPU pv_eoi gfn-cache pinned through SRCU + vCPU refcount saturating; concurrent disable + IRQ delivery cannot wrap.
- **PAX_MEMORY_SANITIZE** — pv_eoi gfn-cache and per-vCPU pending flag zeroed on alloc/free; on WRMSR(0) disable the cached GPA is cleared so a successor enable-MSR cannot inherit it.
- **PAX_UDEREF** — pv_eoi GPA validated against memslots; no raw user pointer dereference.
- **PAX_RAP / kCFI** — `kvm_x86_ops.set_apic_access_page_addr` and `kvm_pv_eoi_set_pending` slots RAP-signed.
- **GRKERNSEC_HIDESYM** — pv_eoi gfn-cache kaddr and per-vCPU pending state redacted unless CAP_SYSLOG + gr-rbac.
- **GRKERNSEC_DMESG** — pv_eoi WRMSR validation failures and lapic-not-in-kernel warnings rate-limited and gated by `dmesg_restrict`.
- **KVM ioctl CAP_SYS_ADMIN strict** — KVM_CAP_PV_EOI advertisement and irqchip creation (KVM_CREATE_IRQCHIP) gated to CAP_SYS_ADMIN + gr-rbac VM-create role.
- **MSR_KVM_PV_EOI_EN passthrough validation** — enable-bit (bit 0) and GPA validated; reserved-bit and out-of-memslot writes intercepted and #GP-injected.
- **lapic_in_kernel required** — MSR_KVM_PV_EOI_EN WRMSR rejected when in-kernel lapic disabled so userspace-lapic VMs cannot enable PV-EOI half-configured.
- **Nested-virt strict** — L2 PV-EOI mediated by L1; L0 pv_eoi cache never directly written by L2 guest.

Per-doc rationale: PV-EOI swaps an IRQ-acknowledge MSR write for a shared flag-byte; the grsec reinforcement here is targeted at (a) GPA validation so a guest cannot redirect the flag write to host-controlled memory, (b) the lapic_in_kernel guard so a half-configured VM cannot enable a broken PV path, and (c) MEMORY_SANITIZE on disable so a successor cannot inherit a stale flag-byte cache.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- LAPIC core (covered in `x86-lapic.md` Tier-3)
- PV clock (covered in `x86-pvclock.md` Tier-3)
- Async-PF (covered in `x86-async-pf.md` Tier-3)
- KVM CPUID (covered in `x86-cpuid.md` Tier-3)
- Implementation code
