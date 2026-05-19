# Tier-3: arch/x86/kvm/x86.c — KVM PV-IPI + PV-TLB-FLUSH + PV-EOI (paravirt fast-path bulk-IPI + cross-vCPU TLB flush)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/x86.c (kvm_pv_send_ipi + kvm_pv_kick_cpu + PV-TLB-flush; see grep "kvm_pv_send_ipi\|MSR_KVM_PV_EOI_EN\|KVM_FEATURE_PV_TLB_FLUSH\|hc_send_ipi")
  - arch/x86/kvm/lapic.c (PV-EOI integration)
  - arch/x86/include/uapi/asm/kvm_para.h (KVM_HC_SEND_IPI / _KICK_CPU; KVM_FEATURE_PV_TLB_FLUSH / _PV_SEND_IPI / _PV_UNHALT)
-->

## Summary

KVM's PV-IPI suite is a collection of paravirt hypercalls that let guest issue per-VM bulk operations as a single VM-exit instead of N exits: KVM_HC_SEND_IPI sends IPI to up to 128 destination LAPICs in one hypercall (vs N writes to ICR-MSR each = N vmexits); KVM_HC_KICK_CPU wakes a halted vCPU after PV-spin-yield (used by qspinlock contention); MSR_KVM_PV_EOI_EN lets guest skip EOI vmexit by writing to a pv_eoi-page; per-VM PV-TLB-FLUSH lets guest issue cross-vCPU TLB-shootdown via a single hypercall. Each saves substantial vmexit cost on multi-vCPU workloads under cross-CPU communication.

This Tier-3 covers PV-IPI / PV-TLB-FLUSH / PV-EOI / PV-KICK_CPU code in `arch/x86/kvm/x86.c` (~600-800 lines) + `arch/x86/kvm/lapic.c` PV-EOI integration.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_pv_send_ipi(kvm, ipi_bitmap_low, ipi_bitmap_high, min, icr, op_64_bit)` | bulk-IPI hypercall | `Kvm::pv_send_ipi` |
| `kvm_pv_kick_cpu_op(kvm, flags, apicid)` | wake halted vCPU | `Kvm::pv_kick_cpu` |
| `KVM_HC_SEND_IPI` (10) | hypercall number | UAPI |
| `KVM_HC_KICK_CPU` (5) | hypercall number | UAPI |
| `KVM_HC_SCHED_YIELD` (11) | yield to specified vCPU | UAPI |
| `MSR_KVM_PV_EOI_EN` (0x4b564d04) | per-vCPU pv_eoi page-addr | UAPI |
| `kvm_lapic_pv_eoi_init(vcpu)` (lapic.c) | per-vCPU PV-EOI init | `Lapic::pv_eoi_init` |
| `apic_set_eoi(apic)` (lapic.c) | EOI delivery (skip if PV-EOI) | `Lapic::set_eoi` |
| `kvm_clear_pv_eoi_pending(vcpu)` | clear pv_eoi pending bit | `Lapic::clear_pv_eoi_pending` |
| `kvm_pv_send_ipi_check_destination(...)` | per-dest validation | `Kvm::pv_send_ipi_validate` |
| `KVM_FEATURE_PV_SEND_IPI` (11) | CPUID feature bit | UAPI |
| `KVM_FEATURE_PV_TLB_FLUSH` (9) | CPUID feature bit | UAPI |
| `KVM_FEATURE_PV_UNHALT` (7) | CPUID feature bit | UAPI |
| `KVM_FEATURE_PV_EOI` (6) | CPUID feature bit | UAPI |
| `KVM_FEATURE_PV_SCHED_YIELD` (13) | CPUID feature bit | UAPI |

## Compatibility contract

REQ-1: KVM_HC_SEND_IPI hypercall:
- Args: a0 = ipi_bitmap_low, a1 = ipi_bitmap_high, a2 = min (apicid offset), a3 = icr (vector + delivery-mode + dest-mode bits).
- 64-bit guest: 64-bit registers; bitmaps cover up to 128 LAPICs from min.
- 32-bit guest: 32-bit; up to 64 LAPICs.
- Implementation: `kvm_pv_send_ipi`:
  1. Iterate set-bits in (low | high << 64).
  2. Per-bit: dest_apicid = min + bit; resolve to vcpu via `kvm_apic_match_dest`.
  3. Compose ICR (matches Intel-LAPIC ICR format): vector + delivery-mode + dest-mode (physical).
  4. `kvm_irq_delivery_to_apic` to dispatch.
  5. count++.
- Return: count of successful deliveries.

REQ-2: KVM_HC_KICK_CPU hypercall:
- Args: a0 = flags (KVM_KICK_CPU_FLAG_*), a1 = target apicid.
- Implementation: `kvm_pv_kick_cpu_op`:
  1. Resolve apicid → vcpu.
  2. kvm_make_request(KVM_REQ_PVCLOCK_GUEST_STOPPED, vcpu) (no — actually KVM_REQ_UNHALT).
  3. kvm_vcpu_kick(vcpu) — IPI-IRQ to wake halted vCPU.

REQ-3: MSR_KVM_PV_EOI_EN setup:
- Guest writes (pv_eoi_page_pa | enable_bit).
- Per-vCPU pv_eoi state:
  - vcpu.arch.pv_eoi.msr_val.
  - vcpu.arch.pv_eoi.cache (gfn_to_hva_cache).
- Per-EOI: if pv_eoi-page-bit-0 = 1 AND interrupt is "uniform-ICR-eligible": guest skips EOI write; KVM observes via apic_set_eoi handler that pv_eoi-bit was 1 and processes EOI without vmexit.

REQ-4: PV-TLB-FLUSH:
- Guest sets per-vCPU PREEMPTED bit in steal-time control-page (KVM_VCPU_FLUSH_TLB) before hypercall.
- KVM observes on next vmenter that flush requested + does TDP/SHADOW MMU TLB-flush before resume.
- Avoids per-CPU IPI to running CPU.

REQ-5: PV-SCHED-YIELD:
- Args: a0 = target apicid.
- Implementation: yield_to(target_vcpu) — let target vCPU run instead of yielding to any.
- Used by qspinlock when waiting on holder.

REQ-6: KVM_FEATURE_PV_SEND_IPI etc CPUID enumeration:
- CPUID 0x40000001:EAX bits per feature.
- Guest probes at boot; uses paravirt or falls back to native.

REQ-7: Per-VM gate via `kvm.arch.pause_in_guest` etc:
- Each PV feature module-param-gated for security-sensitive deployments.
- KVM_CAP_X86_DISABLE_QUIRKS variant.

REQ-8: Hypercall dispatch (in `____kvm_emulate_hypercall`):
1. Read nr from RAX, args from RBX/RCX/RDX/RSI.
2. Validate CPL (typically must be 0; some HCs allow CPL 3).
3. Switch on nr:
   - KVM_HC_SEND_IPI: validate feature enabled; call kvm_pv_send_ipi.
   - KVM_HC_KICK_CPU: validate; kvm_pv_kick_cpu_op.
   - KVM_HC_SCHED_YIELD: yield_to.
   - Other: -KVM_ENOSYS.
4. Set RAX = result.

REQ-9: Per-VM stat counters:
- pv_send_ipi_count, pv_kick_cpu_count, pv_eoi_used_count.
- Visible via /sys/kernel/debug/kvm.

REQ-10: ICR format validation:
- Vector ∈ [16, 255] (vector 0-15 reserved per LAPIC spec).
- Delivery-mode: only Fixed/LowPriority/SMI/NMI/INIT/StartUp valid.
- Dest-mode: physical (PV-IPI uses physical only for simplicity).

REQ-11: PV-EOI atomicity:
- Per-vCPU pv_eoi-page-byte-0 bit-0 = 1 = "uniform delivery; can skip EOI".
- Set by KVM at injection time (if conditions met).
- Cleared by guest at EOI time.
- Race: KVM sets bit during inject; guest clears during processing; pv_eoi pattern allows EOI-skip if bit observed cleared by guest before next inject.

## Acceptance Criteria

- [ ] AC-1: KVM_FEATURE_PV_SEND_IPI advertised via CPUID 0x40000001:EAX bit 11.
- [ ] AC-2: 16-vCPU guest sending broadcast IPI: per-VM only 1 vmexit (the hypercall) instead of 15 separate ICR writes.
- [ ] AC-3: KVM_HC_KICK_CPU: vCPU halted via HLT; another vCPU calls kick; halted vCPU wakes within ~10us.
- [ ] AC-4: PV-EOI: rate-limited interrupt source; per-EOI vmexit count reduced by ~50%.
- [ ] AC-5: PV-TLB-FLUSH: 16-vCPU guest with TLB-shootdown stress; per-flush IPI count reduced (running vCPUs skipped).
- [ ] AC-6: PV-SCHED-YIELD: qspinlock benchmark; throughput increase vs no-yield (test via spinlock-stress).
- [ ] AC-7: kvm-unit-tests `pv_ipi` + `pv_eoi` + `pv_unhalt` pass.
- [ ] AC-8: Spec-violation defense: KVM_HC_SEND_IPI with vector < 16 returns 0 (or -EINVAL); no IPI dispatched.
- [ ] AC-9: Cross-vCPU PV-IPI consistency: bitmap targeting absent apicid silently skipped; counter reflects actual deliveries.

## Architecture

Per-VM additions:

```
struct KvmArch {
  ...
  pv_unhalt: bool,
  pv_lazy_eoi_enabled: bool,
}
```

Per-vCPU additions:

```
struct VcpuArch {
  ...
  pv_eoi: VcpuPvEoi,
  // PV-TLB-flush uses steal-time control-page (separate Tier-3)
}

struct VcpuPvEoi {
  enabled: bool,
  msr_val: u64,
  cache: GfnToHvaCache,
  pending_pv_eoi_pending: AtomicBool,
}
```

`Kvm::pv_send_ipi(kvm, low, high, min, icr, op_64_bit)`:
1. mask := op_64_bit ? (low | (high << 64)) : low.
2. icr_low := icr & 0xFFFFFFFF; icr_high := (icr >> 32).
3. delivery_mode := (icr_low >> 8) & 0x7.
4. vector := icr_low & 0xFF.
5. count := 0.
6. While mask:
   - bit := __ffs(mask).
   - dest_apicid := min + bit.
   - target := kvm_get_vcpu_by_apicid(kvm, dest_apicid).
   - If target:
     - kvm_irq_delivery_to_apic(kvm, NULL, &irq{ .vector=vector, .delivery_mode=delivery_mode, .dest_id=dest_apicid, .dest_mode=APIC_DEST_PHYSICAL }, NULL).
     - count++.
   - mask &= ~(1 << bit).
7. Return count.

`Kvm::pv_kick_cpu(kvm, flags, apicid)`:
1. target := kvm_get_vcpu_by_apicid(kvm, apicid).
2. If !target: return.
3. kvm_make_request(KVM_REQ_UNHALT, target).
4. kvm_vcpu_kick(target).

`Lapic::pv_eoi_init(vcpu)`:
1. Allocate vcpu.arch.pv_eoi.
2. Per-WRMSR(MSR_KVM_PV_EOI_EN):
   - data == 0: disable.
   - else: addr = data & ~1; validate; gfn_to_hva_cache_init.

`Lapic::set_eoi(apic)`:
1. If PV-EOI enabled + bit-0 currently 1 in pv_eoi-page (KVM previously set):
   - Guest already saw "skip EOI" hint; KVM-side EOI bookkeeping (process IRR/ISR transition).
   - Clear pv_eoi-page-bit-0.
2. Else: standard EOI processing.

`Vcpu::handle_msr` MSR_KVM_PV_EOI_EN case:
1. addr := data & ~1; enable := data & 1.
2. If !enable: pv_eoi.enabled = false.
3. Else: validate addr aligned; gfn_to_hva_cache_init; pv_eoi.enabled = true.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pv_send_ipi_bitmap_bounded` | OOB | per-bitmap iteration bounded by 128 (or 64 in 32-bit); apicid range bounded by KVM_MAX_VCPU_IDS. |
| `pv_eoi_page_aligned` | INVARIANT | per-vCPU pv_eoi page 4KiB-aligned. |
| `kick_cpu_apicid_validated` | INVARIANT | apicid resolves to existing vcpu; defense against guest-bogus-apicid causing OOB. |
| `vector_range_validated` | INVARIANT | vector ∈ [16, 255]; defense against reserved-vector causing #GP on inject. |

### Layer 2: TLA+

`virt/kvm/pv_eoi_atomic.tla`:
- States: Disabled, EnabledPendingClear, EnabledPendingSet.
- Transitions per inject (set bit) + EOI (clear bit) + WRMSR (enable/disable).
- Properties:
  - `safety_skip_eoi_only_when_bit_set` — KVM skip-EOI only if pv_eoi-bit observed set at inject time.
  - `safety_no_lost_eoi` — every interrupt receives matching EOI processing (skipped via PV-EOI or vmexit).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Kvm::pv_send_ipi` post: count == popcount(mask ∩ valid-apicids); each delivered IPI matches vector + delivery_mode | `Kvm::pv_send_ipi` |
| `Kvm::pv_kick_cpu` post: target.requests has KVM_REQ_UNHALT; vCPU kicked | `Kvm::pv_kick_cpu` |
| `Lapic::set_eoi` post: PV-EOI page bit cleared if was set; standard ISR/IRR transitions applied | `Lapic::set_eoi` |
| Per-vCPU pv_eoi.enabled iff WRMSR(MSR_KVM_PV_EOI_EN) bit-0 set | `Vcpu::handle_msr` |

### Layer 4: Verus/Creusot functional

`KVM_HC_SEND_IPI(low, high, min, icr) ≡ for each set-bit b in (low | high << 64): native ICR-write to (min+b) with icr-encoded vector + delivery-mode` semantic equivalence: paravirt path produces equivalent observable IPI delivery vs native-emulation path.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

PV-IPI-specific reinforcement:

- **Per-bitmap iteration bounded** — defense against malicious guest setting all bits causing pathological iteration.
- **Vector range [16, 255]** — defense against vector-0-15-reserved causing CPU exception.
- **Delivery-mode whitelist** — defense against undefined modes causing inject failure.
- **Per-feature CPUID gating** — guests only attempt PV-IPI if KVM_FEATURE_PV_SEND_IPI advertised.
- **Per-VM enable flags** — defense against PV-IPI on security-sensitive VMs that need full intercepts.
- **PV-EOI page validated** at WRMSR — defense against guest pointing to host kernel memory.
- **PV-EOI atomic-bit-clear** under apic.lock — defense against torn-update racing inject.
- **kick_cpu validates apicid → real vcpu** — defense against guest kicking non-existent vCPU causing NULL-deref.
- **Hypercall CPL gating** (typically CPL=0) — defense against ring-3 process-bypass attacking hypercall surface.
- **PV-TLB-FLUSH only flushes guest-managed TLB** — defense against host-IPI cross-VM cache leak.
- **PV-SCHED-YIELD validates target apicid** — defense against yield_to invalid task.
- **Per-VM stat counters bounded** — defense against attacker monitoring counter overflow timing.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — hypercall argument vectors (PV-IPI APIC-id bitmap, PV-TLB-FLUSH GFN list) bounded by per-hypercall length-limit; copy length validated against `KVM_MAX_VCPUS` for IPI bitmaps.
- **PAX_KERNEXEC** — `kvm_pv_ipi`, `kvm_pv_flush_tlb`, `kvm_pv_send_ipi_to_cpu` resolve through RX-only kernel text.
- **PAX_RANDKSTACK** — hypercall entry path inherits RANDKSTACK from VM-exit.
- **PAX_REFCOUNT** — per-vCPU and per-VM hypercall serial / kick counters saturating; concurrent send-ipi + vCPU destroy cannot wrap.
- **PAX_MEMORY_SANITIZE** — per-hypercall scratch buffers zeroed on allocation; PV-IPI APIC bitmap zeroed between calls so prior recipient set never bleeds.
- **PAX_UDEREF** — hypercall args fetched from guest registers (already in kernel `pt_regs`-like struct); no user-pointer follow.
- **PAX_RAP / kCFI** — hypercall dispatch table (`hypercall_table` / `kvm_emulate_hypercall`) RAP-signed.
- **GRKERNSEC_HIDESYM** — per-VM hypercall stats kaddrs redacted unless CAP_SYSLOG + gr-rbac.
- **GRKERNSEC_DMESG** — invalid-hypercall, target-apicid-mismatch warnings rate-limited and gated by `dmesg_restrict`.
- **KVM ioctl CAP_SYS_ADMIN strict** — KVM_CAP_PV_IPI / KVM_CAP_PV_SEND_IPI advertisement gated to CAP_SYS_ADMIN + gr-rbac VM-create role.
- **Hypercall CPL gating** — PV-IPI / PV-SCHED-YIELD / PV-TLB-FLUSH allowed from CPL=0 only; ring-3 ring-bypass hypercall rejected with #GP.
- **Target apicid validation** — `kick_cpu` and PV-IPI target apicid validated against in-kernel APIC map; non-existent vCPU rejects rather than NULL-deref.
- **Cross-VM containment** — PV-IPI bitmap can only address current-VM vCPUs; cross-VM kick rejected at lookup.
- **Nested-virt strict** — L2 hypercalls trapped through L1; L0 PV-IPI never delivers cross-L1 IPIs from a malicious L2.

Per-doc rationale: PV-IPI / PV-TLB-FLUSH replace IPI loops with a hypercall that touches per-vCPU state across the VM; the grsec reinforcement here keeps the target-apicid validated to prevent NULL-deref, gates the hypercalls at CPL=0 so ring-3 guest processes cannot reach the hypercall surface, and enforces cross-VM containment so a malicious tenant cannot kick a co-resident VM's vCPU.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- VMX/SVM (covered in `x86-vmx.md` / `x86-svm.md` Tier-3s)
- kvmclock (covered in `x86-pvclock.md` Tier-3)
- async-pf (covered in `x86-async-pf.md` Tier-3)
- steal-time (covered in `x86-steal-time.md` Tier-3)
- Hyper-V hypercall surface (covered in `x86-hyperv.md` future Tier-3)
- Implementation code
