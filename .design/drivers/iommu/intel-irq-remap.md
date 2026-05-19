# Tier-3: drivers/iommu/intel/irq_remapping.c — Intel interrupt remapping (IRTE table + posted-IRQ + per-IOMMU IR cap)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/intel-iommu.md
upstream-paths:
  - drivers/iommu/intel/irq_remapping.c
  - drivers/iommu/intel/irq_remapping.h
  - include/linux/intel-iommu.h (IR cap bits)
  - drivers/iommu/intel/dmar.c (cap detection)
-->

## Summary

Intel Interrupt Remapping (IR) is the VT-d feature that puts an IRTE (Interrupt Remap Table Entry) lookup between every device-issued MSI/MSI-X and the LAPIC. Each IRTE is an 128-bit entry in a per-IOMMU IRT table; per-MSI-message becomes a 16-bit IRTE-handle that the IOMMU translates to {dest-LAPIC, vector, delivery-mode}. Critical for: x2APIC required (no IR → x2APIC blocked by spec); KVM posted-interrupts (IRTE.posted=1 → IOMMU writes guest-PIR directly bypassing host); PCI hot-plug security (per-device IRTE validates source-id, blocking spoofed MSI from rogue device).

This Tier-3 covers `drivers/iommu/intel/irq_remapping.c` (~1605 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct intel_ir_data` | per-IRQ IR data attached to irq_data | `drivers::iommu::intel::IrData` |
| `struct irte` | 128-bit IRTE bit layout | `Irte` |
| `intel_setup_ioapic_entry(...)` | IOAPIC entry → IRTE-handle remap | `IrData::setup_ioapic_entry` |
| `intel_irq_remapping_alloc(...)` | per-MSI alloc of IRTE-handle | `IrData::alloc` |
| `intel_irq_remapping_free(...)` | per-MSI free | `IrData::free` |
| `intel_irq_remapping_activate(...)` | activate IRTE for IRQ | `IrData::activate` |
| `intel_irq_remapping_deactivate(...)` | deactivate | `IrData::deactivate` |
| `intel_set_affinity(...)` | per-IRQ migrate destination | `IrData::set_affinity` |
| `intel_setup_irq_remapping(iommu)` | per-IOMMU init IR (alloc IRT, prog reg) | `Iommu::setup_ir` |
| `intel_disable_irq_remapping()` | per-IOMMU disable IR | `Iommu::disable_ir` |
| `intel_irq_remap_add_device(info)` | per-device IR-domain attach | `IrData::add_device` |
| `intel_compose_msi_msg(...)` | per-MSI compose into IRTE-handle msg | `IrData::compose_msi_msg` |
| `modify_irte(handle, irte)` | atomic IRTE update | `IrtTable::modify_irte` |
| `qi_flush_iec(iommu, index, mask)` | invalidate IEC (interrupt-entry-cache) | `Iommu::flush_iec` |
| `set_irte_pid(iommu, index, pid)` (posted-IRQ) | mod IRTE for posted-interrupt | `IrtTable::set_irte_pid` |
| `intel_post_msi_msg(...)` | KVM posted-IRQ entry path | `IrData::post_msi_msg` |
| `set_msi_sid(irte, dev)` | source-id program | `Irte::set_sid` |

## Compatibility contract

REQ-1: Per-IOMMU IRT table:
- `irte[N]` array of 128-bit entries; N typically 64K (16-bit handle space).
- Allocated as IOMMU-mapped pages on init; physical addr programmed into IOMMU IRT_REG (MMIO offset 0xB8).
- Per-IRTE flags: present, dest-mode, redir-hint, dest-id, vector, delivery-mode, source-id (BDF + source-validation-type).

REQ-2: Per-IRQ alloc flow:
- Linux MSI alloc requests IRQ from intel-IR-domain.
- `IrData::alloc(domain, virq, nr_irqs, info)`:
  - Per-IRQ: alloc IRTE handle from per-IOMMU bitmap.
  - Allocate `IrData`; bind virq → IrData → IRTE-handle.
  - Populate IRTE: dest-id from virq affinity; vector from IRQ vector; delivery-mode from MSI config; source-id from PCI BDF.
  - `modify_irte(handle, &irte)`: atomic-write IRTE (cmpxchg128 or qi_flush_iec round-trip).

REQ-3: Source-ID validation (Source-Validation-Type field in IRTE):
- SVT=0: no source validation.
- SVT=1: validate source-id == BDF (require {req_id == sid}).
- SVT=2: validate source-id == BDF range (for PCI bridge).
- Defense against rogue device spoofing MSI from another device's BDF.

REQ-4: x2APIC ↔ IR coupling: x2APIC enable requires IR active per Intel spec (else interrupts silently misdelivered).

REQ-5: Per-IRQ affinity migration:
- `IrData::set_affinity(data, mask, force)`:
  - Compute new dest-id from mask.
  - `modify_irte(handle, &irte)` with new dest-id.
  - `qi_flush_iec(iommu, index, 0)` invalidates IEC cache so next MSI delivery sees new IRTE.
  - Wait for in-flight: cmpxchg + retry until pending bit clear.

REQ-6: Posted-IRQ support (KVM integration):
- IRTE.posted bit + posted-descriptor-addr field (PDA).
- `set_irte_pid(iommu, index, pid)`: mod IRTE so MSI from device → IOMMU writes guest-PIR (Posted-Interrupt-Request) bitmap directly.
- IOMMU sets pending bit + sends Notification-Vector to LAPIC.
- Bypasses host LAPIC entirely → reduced VM-Exit pressure for SR-IOV.

REQ-7: Per-IOMMU IR cap detection (in `dmar.c` cap parse, gated by ECAP.IR=1):
- `intel_irq_remap_capable(iommu)` checks ECAP[3]=1.
- Subsequent `setup_ir` allocates IRT + programs IRTE_REG.

REQ-8: IRTE invalidation:
- Per-IRTE update: must invalidate per-IRTE entry in IEC via QI command.
- QI desc layout: type=IEC, index, mask.
- `qi_submit_sync(...)` waits for completion via WAIT desc.

REQ-9: Per-IRQ IO-APIC redirect-table integration:
- For non-MSI legacy IRQs: IO-APIC RTE programmed with IRTE-handle (16-bit) instead of dest-vector.
- `intel_setup_ioapic_entry`: per-RTE compose IRTE-handle from IRQ-config.
- IO-APIC delivers MSI message containing IRTE-handle to IOMMU.

REQ-10: Per-HPET MSI: same IRTE path; HPET MSI events go via IOMMU.

REQ-11: Compose MSI msg with IRTE-handle:
- 32-bit MSI addr: format-bit + IRTE-handle[14:0] + handle-bit.
- IOMMU recognizes handle format-bit + dispatches to IRT lookup.
- IRTE.dest_id + vector → final LAPIC delivery.

## Acceptance Criteria

- [ ] AC-1: Boot Intel x86_64 with `intel_iommu=on` + IR-cap host: `dmesg | grep "Enabled IRQ remapping"` per-IOMMU.
- [ ] AC-2: x2APIC + IR test: `dmesg | grep "x2APIC: Enabled"` after IR active; lscpu shows x2apic.
- [ ] AC-3: PCI MSI device: NIC + NVMe MSI-X allocated via intel-IR-domain; `cat /proc/interrupts | grep dmar` shows IR-domain entry.
- [ ] AC-4: IRQ affinity migrate: `echo 4 > /proc/irq/N/smp_affinity` for an MSI IRQ; subsequent MSI delivery to CPU4; `qi_flush_iec` traced.
- [ ] AC-5: Source-ID-validation enforcement: spoofed MSI from rogue source-id (test via DMA fuzzer) ignored; IOMMU fault logged.
- [ ] AC-6: Posted-IRQ KVM test: SR-IOV passthrough VM with vfio-pci; KVM_CAP_INTEL_PI active; per-MSI delivered via posted-descriptor; reduced VM-exits observed.
- [ ] AC-7: IRTE update atomicity: IRQ-affinity-migrate during heavy MSI traffic; no spurious-IRQ on previous CPU.
- [ ] AC-8: kvm-unit-tests / vfio-test posted-interrupt suite passes.

## Architecture

`Iommu` per-IOMMU adds IR fields:

```
struct Iommu {
  ...
  ir_table: Option<KArc<IrtTable>>,
  ir_supported: bool,                       // ECAP.IR
  posted_irq_supported: bool,               // ECAP.PI
}

struct IrtTable {
  base: KBox<[Irte; 65536]>,                // 1MiB IRT per IOMMU
  base_phys: u64,                            // physical addr programmed to IRT_REG
  bitmap: BitmapAlloc,                       // free-IRTE-handle bitmap
  qi: KArc<QueuedInvalidation>,              // ref to per-IOMMU QI
}

struct Irte {                                // 128-bit packed
  present: u1,
  dest_mode: u1,
  redir_hint: u1,
  trigger_mode: u1,
  dest_id: u32,
  vector: u8,
  delivery_mode: u3,
  pst: u1,                                   // posted bit
  source_id: u16,
  sq: u2,                                    // source-id qualifier
  svt: u2,                                   // source-validation-type
  // when pst=1:
  pda_low: u26,
  pda_high: u26,
  urg: u1,
  notif_vector: u8,
  ...
}

struct IrData {
  iommu: KArc<Iommu>,
  irte_index: u16,
  irq_2_iommu: Irq2Iommu,
  msi_addr: u32,
  msi_data: u32,
  prev_ga: Option<PostedGa>,                 // KVM posted-IRQ link
}
```

`Iommu::setup_ir` flow:
1. Allocate IRT (1MiB; 64K entries × 16 bytes).
2. Pin to physical addr; program IOMMU IRT_REG = base_phys | size_bits.
3. Per-IOMMU enable IRE bit (Interrupt-Remap-Enable) via GCMD register.
4. Wait for GSTS.IRES bit set (IR-active confirm).
5. Init `bitmap` for handle alloc.

`IrData::alloc` (per-MSI):
1. Allocate IRTE-handle from `iommu.ir_table.bitmap`.
2. Allocate IrData; populate.
3. Construct IRTE:
   - present=1; dest_id=cpu_cpumask_to_apicid(affinity); vector=allocated_vector;
   - delivery_mode=fixed (or low-pri); source-id=PCI BDF; svt=1 (BDF-match).
4. `IrtTable::modify_irte(handle, &irte)`:
   - cmpxchg128 atomic on irte.
   - If hardware lacks cmpxchg128 atomicity guarantees: write low 64 first with present=0; write high 64; write low with present=1.
   - `Iommu::flush_iec(handle)` → QI desc submit → wait for IWC.

`IrData::compose_msi_msg(msg)`:
- msg.addr_lo = MSI_ADDR_BASE_LO | (handle << 5) | format_bit | sub_handle_bit.
- msg.addr_hi = MSI_ADDR_BASE_HI.
- msg.data = 0 (vector embedded in IRTE, not data).

`IrData::set_affinity(mask, force)`:
1. Compute new dest_id from mask.
2. Read current IRTE.
3. Update dest_id field.
4. `modify_irte(handle, &updated)`.
5. `flush_iec(handle)`; wait for QI completion.

`IrData::post_msi_msg(...)` (KVM posted-IRQ activate):
1. KVM passes posted-descriptor physical addr (PDA) + notification vector.
2. Update IRTE: pst=1; pda_low=PDA[31:6]; pda_high=PDA[63:32]; notif_vector=NV.
3. `modify_irte(handle, &updated)` + `flush_iec`.
4. Subsequent MSI from device → IOMMU writes PIR-bitmap @ PDA + sends NV to LAPIC; LAPIC injects guest IRQ on next vmenter.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `irte_handle_no_oob` | OOB | per-IRTE handle ∈ [0, 65535]; bitmap-alloc enforces. |
| `irt_table_no_oob` | OOB | per-IRTE-write to ir_table.base[handle] bounded by 65536. |
| `irte_atomic_update` | INVARIANT | IRTE update via cmpxchg128 atomic OR present-bit-clear-then-set sequence; defense against torn IRTE causing wrong delivery. |
| `posted_pda_aligned` | INVARIANT | per-IRTE.pda 64-byte aligned (Intel spec); defense against unaligned PDA causing #GP on IOMMU. |

### Layer 2: TLA+

`drivers/iommu/intel/irte_consistency.tla` (already exists from earlier Tier-3) models per-IRTE atomic update + flush_iec sequence.

`drivers/iommu/intel/irq_remap_alloc.tla`: per-IRTE-handle alloc state ∈ {Free, Allocated, Active, Migrating, Released}; transitions per IrData lifecycle.

Properties:
- `safety_no_active_after_release` — once Released, no Active without re-alloc.
- `safety_atomic_migrate` — Migrating state always followed by either Active(new-cpu) or rollback to Active(old-cpu); never Free.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IrData::alloc` post: handle in [0, 65535]; IRT[handle] populated; bitmap bit set | `IrData::alloc` |
| `IrData::set_affinity` post: IRTE[handle].dest_id == apicid(mask); IEC flushed | `IrData::set_affinity` |
| `IrData::free` post: bitmap bit clear; IRT[handle].present == 0 | `IrData::free` |
| Per-IRQ at most one active IRTE-handle | `IrData::activate` |
| IRTE.svt == 1 ∧ source_id == device-BDF for all device-MSI IRTEs | `Irte::set_sid` |

### Layer 4: Verus/Creusot functional

`IrData::alloc(virq) → device MSI generates LAPIC IPI to dest-cpu` round-trip equivalence: per-MSI from device with handle-encoded addr → IOMMU IRT lookup → LAPIC IPI to IRTE.dest_id with IRTE.vector.

## Hardening

(Inherits row-1 features from `drivers/iommu/intel-iommu.md` § Hardening.)

intel-IR-specific reinforcement:

- **Source-ID validation (SVT=1)** by default for all device IRTEs — defense against rogue-device MSI spoofing.
- **IRT in IOMMU-protected memory** — defense against device-DMA-write to IRT (would require IOMMU-bypass).
- **Per-IRTE atomic-update via cmpxchg128 OR present-clear-flush-set** — defense against torn-IRTE interim state.
- **`flush_iec` after every IRTE update** — defense against stale IEC cache causing in-flight MSI mis-delivery.
- **x2APIC gated on IR-active** — defense against x2APIC + non-IR causing silent IRQ mis-delivery.
- **PCI hotplug add-device IRTE-late-bind** — IRTE allocated only after PCI add-device + driver attach; defense against pre-attach IRTE leaks.
- **Per-IRTE-handle bitmap held under iommu_lock** — defense against concurrent alloc-free races.
- **Posted-IRQ PDA validated to be guest-physical-page-aligned + within KVM memslot** — defense against malicious guest PDA pointing to host kernel memory.
- **Per-IRQ free flushes IEC + clears IRTE.present** — defense against late-MSI delivery to freed virq.
- **IRT_REG write-once at boot** — defense against runtime IRT-base relocation causing in-flight MSI mis-delivery.

## Grsecurity/PaX-style Reinforcement

Hardened-policy supplement layered above the baseline `## Hardening`. Rookery treats Intel interrupt remapping as a system-trust anchor for MSI delivery; any compromise of IRTE entries gives device-issued IPIs the ability to target arbitrary LAPIC/vector pairs, so PaX/grsec-equivalent mitigations apply broadly.

- **PAX_USERCOPY** on any IRTE-handle / source-id leaked to user via `/proc/interrupts` formatting.
- **PAX_KERNEXEC** on IR allocator, `modify_irte`, and QI submission paths (RO post-init).
- **PAX_RANDKSTACK** on MSI-alloc/free entry chains crossing PCI hotplug.
- **PAX_REFCOUNT** on `IrData`, `IrtTable`, posted-IRQ GA-handles to catch refcount overflow into IRTE leaks.
- **PAX_MEMORY_SANITIZE** zeroes IRT pages on alloc and IRTE entries on free (`present=0` + scrub).
- **PAX_UDEREF** on copy_from_user paths for IRQ-affinity sysfs writes.
- **PAX_RAP/kCFI** on the IR-domain irq_chip vtable (`activate`, `set_affinity`, `compose_msi_msg`) to forbid forged callbacks.
- **GRKERNSEC_HIDESYM** hides `intel_ir_table_base` / `iommu->ir_table` from `/proc/kallsyms` and crash dumps.
- **GRKERNSEC_DMESG** restricts DMAR/IR fault printouts to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict** on `intel_iommu=` reconfiguration and any debugfs IR controls.
- **GRKERNSEC_DMA strict-mode** default — devices behind disabled IR remain blocked from MSI delivery.
- **SVT=1 mandatory** under hardened policy: SVT=0 IRTEs refused even for legacy compat shims.
- **DMAR ACPI signature verify** before trusting ECAP.IR/PI bits; mismatched checksums disable IR rather than fall back.
- **Posted-IRQ PDA** validated against KVM memslot + nested-userns boundary; nested guests barred from PI by default.
- **VT-d IRT_REG write-once-locked** after boot via IOMMU register-write capability gate.

Rationale: with IR active, an attacker who can corrupt an IRTE owns kernel control flow via crafted MSI vectors. Hardened Rookery treats IRT memory, IEC invalidation, and source-id validation as TCB-critical and refuses degraded modes (no SVT, no flush) that upstream tolerates for compatibility.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Generic IOMMU API (covered in `iommu-core.md` Tier-3)
- Intel IOMMU translation domain ops (covered in `intel-iommu.md` Tier-3)
- DMAR ACPI parse (covered in `intel-dmar.md` Tier-3)
- KVM posted-interrupt (covered in `virt/kvm/x86-posted-irq.md` future Tier-3)
- AMD AVIC interrupt remap (different vendor)
- Implementation code
