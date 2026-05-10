# Tier-3: arch/x86/kvm/vmx/vmx.c (PML subset) — Intel Page-Modification-Logging for KVM dirty-tracking

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/dirty-tracking.md
upstream-paths:
  - arch/x86/kvm/vmx/vmx.c (PML subset: ~130 (module-param), ~4754, ~4919..4920, ~6387..6418 vmx_flush_pml_buffer, ~6687, ~7677..7708, ~7818, ~8379, ~8729)
  - arch/x86/include/asm/vmx.h (PML_ADDRESS / PML_INDEX VMCS fields)
-->

## Summary

Intel Page-Modification-Logging (PML; Broadwell+; SDM-Vol3 §29.4) is hardware-assisted dirty-page tracking. Per-vCPU PML buffer (single 4KB page = 512 entries × 8B GPA). On EPT-PTE write fault that turns A+D bits → CPU records the GFN in PML buffer at PML_INDEX. PML_INDEX wraps from 0x1FF down; when overflow → PML-FULL vmexit. KVM `vmx_flush_pml_buffer` walks per-buffer entries, marks GFNs dirty in dirty-bitmap or pushes to dirty-ring (replacing software-only D-bit walking). Per-VM gated by `enable_pml` module param + at least one memslot with KVM_MEM_LOG_DIRTY_PAGES. Critical for: low-overhead live-migration dirty-tracking on supported hardware.

This Tier-3 covers PML subset of `vmx.c` (~150 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enable_pml` (module-param) | per-host enable | `VmxOps::enable_pml` |
| `vmx->pml_pg` | per-vCPU PML buffer page | `VmxVcpu::pml_pg` |
| `PML_ADDRESS` (vmcs field 0x200E) | per-vCPU PML buffer phys-addr | UAPI |
| `PML_INDEX` (vmcs field 0x812) | per-vCPU PML write index | UAPI |
| `PML_LOG_NR_ENTRIES` (512) | per-buffer entry cap | shared |
| `vmx_flush_pml_buffer()` | per-vCPU drain → bitmap/ring | `Vmx::flush_pml_buffer` |
| `vmx_get_pml_log_dirty_pages()` | per-vCPU GFN-walk callback | `Vmx::get_pml_log_dirty_pages` |
| `cpu_has_vmx_pml()` | per-host capability | `VmxOps::cpu_has_pml` |
| `vmx_setup_l2_pml()` | per-nested L2 PML setup | `Vmx::setup_l2_pml` |
| `kvm_arch_log_dirty_log_buffer` | per-arch dirty-bitmap update | `KvmArch::log_dirty_log_buffer` |
| `enable_log_dirty_pt_masked` | per-mmu wp-restore on flush | shared |

## Compatibility contract

REQ-1: Per-host capability detection:
- `cpu_has_vmx_pml`: VMX_BASIC.PML capability bit.
- Module-param `enable_pml` defaults to true if available.

REQ-2: Per-vCPU PML buffer:
- Single 4KB page allocated at vmx_vcpu_create.
- vmx.pml_pg = alloc_page(GFP_KERNEL_ACCOUNT | __GFP_ZERO).
- Per-vmenter: vmcs.PML_ADDRESS = page_to_phys(pml_pg).
- Per-vmenter: vmcs.PML_INDEX = 0x1FF (start full).

REQ-3: HW behavior (per-write-fault on EPT-PTE):
- HW writes (GPA & ~0xFFF) to PML[PML_INDEX].
- Decrement PML_INDEX.
- If PML_INDEX wrapped past 0 (i.e. == 0xFFFF after dec): vmexit with reason PML_FULL.

REQ-4: vmx_flush_pml_buffer flow:
- Per-vmexit (any) or per-PML-FULL: KVM walks pml_pg entries.
- For idx in (PML_INDEX+1)..512:
  - gfn = pml_buf[idx] >> PAGE_SHIFT.
  - kvm_vcpu_mark_page_dirty(vcpu, gfn).
- Reset PML_INDEX = 0x1FF.

REQ-5: Per-VM gating:
- vmcs.execution-controls bit ENABLE_PML (secondary control).
- Set iff enable_pml ∧ kvm.nr_memslots_dirty_logging > 0.

REQ-6: Per-nested L2:
- vmx_setup_l2_pml: L0 emulates PML for L2 by combining vmcs02.

REQ-7: Per-VM dirty_log_pt_masked:
- After flush, restore PTE write-protection so subsequent writes re-fault.

REQ-8: Per-VM destroy:
- Free pml_pg via __free_page.

REQ-9: Module-param disable:
- enable_pml=0 at module load: PML disabled; software dirty-tracking used.

REQ-10: PML + APICv coexistence:
- PML and APICv compatible.
- PML and shadow MMU compatible.

## Acceptance Criteria

- [ ] AC-1: Boot KVM with PML-capable CPU + dirty-log enabled: enable_pml=1; vmcs.PML_ADDRESS set.
- [ ] AC-2: Guest write to write-protected page: HW pushes GPA to PML buffer.
- [ ] AC-3: PML_INDEX wraps to 0xFFFF: PML_FULL vmexit.
- [ ] AC-4: vmx_flush_pml_buffer walks 512 entries: per-GFN marked dirty.
- [ ] AC-5: Per-VM dirty-log disabled: ENABLE_PML cleared in vmcs.
- [ ] AC-6: Module-param enable_pml=0: PML disabled.
- [ ] AC-7: Per-VM destroy: pml_pg freed.
- [ ] AC-8: Live-migration: PML-tracked dirty pages copied; subsequent writes re-tracked.
- [ ] AC-9: Per-vCPU pause + resume: PML_INDEX restored.
- [ ] AC-10: Per-PML-FULL vmexit: handled; buffer flushed; vmenter resumes.
- [ ] AC-11: kvm-unit-tests `pml` test passes.

## Architecture

Per-vCPU PML state:

```
struct VmxVcpu {
  ...
  pml_pg: Box<[u64; PML_LOG_NR_ENTRIES]>,        // 4KB page, 512 × 8B
}

const PML_LOG_NR_ENTRIES: usize = 512;
```

Per-host:

```
struct VmxOps {
  enable_pml: bool,
  pml_log_nr_entries: u32,                       // 512
}
```

`Vmx::cpu_has_pml() -> bool`:
1. Return VMX_SECONDARY_CTLS & ENABLE_PML.

`VmxOps::vcpu_create_pml(vmx) -> Result<()>`:
1. If !enable_pml: return Ok.
2. vmx.pml_pg = alloc_page(GFP_KERNEL_ACCOUNT | __GFP_ZERO).
3. If !vmx.pml_pg: return Err(NoMem).
4. Ok.

`VmxOps::vcpu_load_pml(vcpu)` (at vmenter prep):
1. If vmx.pml_pg ∧ vmx_dirty_log_enabled(vcpu.kvm):
   - vmcs.SECONDARY_VM_EXEC_CONTROL |= SECONDARY_EXEC_ENABLE_PML.
   - vmcs.PML_ADDRESS = page_to_phys(vmx.pml_pg).
   - vmcs.PML_INDEX = PML_LOG_NR_ENTRIES - 1.
2. Else:
   - vmcs.SECONDARY_VM_EXEC_CONTROL &= !SECONDARY_EXEC_ENABLE_PML.

`Vmx::flush_pml_buffer(vcpu)`:
1. pml_idx = vmcs.PML_INDEX.
2. If pml_idx == 0xFFFF: pml_idx = 0  // wrap.
3. Else: pml_idx++  // first valid entry.
4. For idx in pml_idx..PML_LOG_NR_ENTRIES:
   - gpa = pml_pg[idx].
   - gfn = gpa >> PAGE_SHIFT.
   - kvm_vcpu_mark_page_dirty(vcpu, gfn).
5. vmcs.PML_INDEX = PML_LOG_NR_ENTRIES - 1.

`VmxOps::vcpu_free_pml(vmx)`:
1. If vmx.pml_pg: __free_page(vmx.pml_pg); vmx.pml_pg = NULL.

`Vmx::setup_l2_pml(vmx)`:
1. /* Per-nested: L0 sets up PML for L2; copies L0 pml_pg into vmcs02. */
2. If is_guest_mode(vcpu) ∧ ENABLE_PML in vmcs12: vmcs02.PML_ADDRESS = page_to_phys(vmx.pml_pg).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pml_pg_4k_aligned` | INVARIANT | per-vCPU pml_pg phys-addr 4KB-aligned. |
| `pml_index_in_range` | INVARIANT | per-vmenter: vmcs.PML_INDEX ∈ [0, 511] ∨ wrap-marker. |
| `pml_address_set_iff_enable_pml` | INVARIANT | vmcs.SECONDARY_EXEC_ENABLE_PML ⟺ vmcs.PML_ADDRESS valid. |
| `flush_drains_buffer` | INVARIANT | post-flush: PML_INDEX = 511. |
| `pml_pg_lifetime_eq_vcpu` | INVARIANT | pml_pg lifetime ≤ vCPU lifetime. |

### Layer 2: TLA+

`virt/kvm/pml.tla`:
- Per-vCPU HW push + per-vmexit flush + per-PML-FULL exit.
- Properties:
  - `safety_no_lost_dirty_page` — per-write to WP-page ⟹ GPA in PML or already-dirty.
  - `safety_no_double_dirty` — per-flushed entry: dirty-bitmap-set; PML cleared.
  - `liveness_full_eventually_flushed` — per-PML-FULL ⟹ vmexit + flush.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vmx::flush_pml_buffer` post: per-entry GFN marked dirty; PML_INDEX reset | `Vmx::flush_pml_buffer` |
| `VmxOps::vcpu_create_pml` post: pml_pg allocated when enable_pml | `VmxOps::vcpu_create_pml` |
| `VmxOps::vcpu_load_pml` post: vmcs.PML_ADDRESS / INDEX correct iff dirty-log on | `VmxOps::vcpu_load_pml` |
| `VmxOps::vcpu_free_pml` post: pml_pg freed; vmx.pml_pg=NULL | `VmxOps::vcpu_free_pml` |

### Layer 4: Verus/Creusot functional

`Per-WP-page write → HW pushes GPA to PML[PML_INDEX]; PML_INDEX-- ; PML_FULL on wrap → vmexit → flush_pml_buffer → kvm_vcpu_mark_page_dirty per-GFN` semantic equivalence: per-Intel SDM §29.4 PML spec.

## Hardening

(Inherits row-1 features from `virt/kvm/dirty-tracking.md` § Hardening.)

PML-specific reinforcement:

- **Per-pml_pg locked in host phys-addr** — defense against host swap moving pml-page mid-vmexit.
- **Per-VM gated by nr_memslots_dirty_logging** — defense against unnecessary PML overhead when not needed.
- **Per-flush always runs at vmexit when dirty-log on** — defense against per-vmexit lost dirty info.
- **Per-PML-FULL vmexit handled** — defense against HW infinite-FULL loop.
- **Per-PML_INDEX wrap-marker handled** — defense against per-flush off-by-one.
- **Per-nested L2: L0 owns PML setup** — defense against L1 spoofing PML_ADDRESS.
- **Per-module-param enable_pml runtime gating** — defense against per-host CPU bug.
- **Per-EPT-WP restore after flush** — defense against per-flushed-page subsequent write missed.
- **Per-vCPU pml_pg zero-init** — defense against per-init stale GPA leaking to dirty-bitmap.
- **Per-VM destroy frees pml_pg** — defense against per-VM-leak.
- **Per-PML capability check at module load** — defense against per-non-PML-CPU mis-enable.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM dirty tracking (covered in `dirty-tracking.md` Tier-3)
- KVM dirty ring (covered in `dirty-ring.md` Tier-3)
- KVM EPT MMU (covered in `x86-mmu-tdp.md` Tier-3)
- KVM live migration (covered separately)
- Implementation code
