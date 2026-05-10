# Tier-3: arch/x86/kvm/vmx/vmx.c (VPID subset) — VPID (Virtual Processor ID) per-VM TLB tagging + INVVPID

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-vmx.md
upstream-paths:
  - arch/x86/kvm/vmx/vmx.c (VPID-related sections; see grep "vpid|VPID|invvpid|INVVPID")
  - arch/x86/kvm/vmx/nested.c (nested-VPID)
  - arch/x86/include/uapi/asm/vmx.h
-->

## Summary

VPID (Virtual Processor ID) is an Intel-VMX feature that tags each TLB entry with a 16-bit VPID identifier so the CPU can keep VM-A's TLB entries valid even after switching to VM-B (vs without VPID, every VMENTER flushes TLB). KVM allocates per-vCPU unique VPID from a per-host bitmap (16384 IDs = VMX_NR_VPIDS), populates vmcs.virtual_processor_id, and uses INVVPID to selectively flush per-VPID TLB on guest CR3 change, MOV-to-CR3, etc. Substantial perf benefit: cross-VM context switches no longer suffer full-TLB-flush penalty (~10000+ cycles per cross-switch).

This Tier-3 covers VPID-related code in `vmx.c` (~200-300 lines) + nested-VPID in nested.c.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `vmx_vpid_bitmap` | per-host VPID alloc bitmap | `Vmx::VPID_BITMAP` |
| `vmx_vpid_lock` | bitmap lock | `Vmx::VPID_LOCK` |
| `enable_vpid` | module param | `Vmx::ENABLE_VPID` |
| `allocate_vpid()` | per-vCPU alloc from bitmap | `Vmx::allocate_vpid` |
| `free_vpid(vpid)` | per-vCPU release | `Vmx::free_vpid` |
| `vmx_invvpid(...)` | INVVPID instruction wrapper | `Vmx::invvpid` |
| `vmx_flush_tlb_user(vcpu)` | TLB flush per-vCPU (uses INVVPID) | `VcpuVmx::flush_tlb_user` |
| `vmx_flush_tlb_all(vcpu)` | full TLB flush | `VcpuVmx::flush_tlb_all` |
| `vmx_flush_tlb_guest(vcpu)` | guest-only TLB flush | `VcpuVmx::flush_tlb_guest` |
| `vmx_flush_tlb_current(vcpu)` | per-vCPU current flush | `VcpuVmx::flush_tlb_current` |
| `nested_vmx_change_vpid_to_vmcs02_vpid(...)` (nested.c) | per-L2 VPID assign | `VmxNested::assign_vpid` |
| `vpid_sync_context(vpid)` | per-VPID flush helper | `Vmx::vpid_sync_context` |
| `vpid_sync_vcpu_addr(vpid, addr)` | per-VPID per-addr selective flush | `Vmx::vpid_sync_vcpu_addr` |
| `vpid_sync_vcpu_global()` | global VPID flush (typically only at VMX-disable) | `Vmx::vpid_sync_global` |

## Compatibility contract

REQ-1: VPID feature-detection:
- Per-host: MSR_IA32_VMX_PROCBASED_CTLS2 bit 5 (enable VPID) AND VMX-EPT-VPID-CAP bits.
- VMX_NR_VPIDS = 16384 (per-Intel-spec 14-bit unique IDs; 0 reserved for "no-tagging").

REQ-2: Per-host VPID bitmap:
- Static `DECLARE_BITMAP(vmx_vpid_bitmap, VMX_NR_VPIDS)`.
- Bit-set = VPID in use; bit-clear = available.
- Protected by `vmx_vpid_lock` (spinlock).

REQ-3: Per-vCPU VPID alloc (`allocate_vpid`):
1. Acquire vmx_vpid_lock.
2. vpid := find_first_zero_bit(&vmx_vpid_bitmap, VMX_NR_VPIDS).
3. If vpid == VMX_NR_VPIDS: return 0 (none-available; defaults to no-VPID-tagging).
4. set_bit(vpid, &vmx_vpid_bitmap).
5. Release lock.
6. Return vpid.

REQ-4: Per-vCPU VPID release:
1. If vpid == 0: skip.
2. Acquire vmx_vpid_lock.
3. clear_bit(vpid, &vmx_vpid_bitmap).
4. Release lock.

REQ-5: VMCS VPID setup at vCPU init:
1. vpid := allocate_vpid().
2. vmcs.virtual_processor_id = vpid.
3. vmcs.secondary_exec_control |= ENABLE_VPID.
4. INVVPID-context for new vpid (no-stale-entries).

REQ-6: INVVPID instruction:
- `invvpid` with 4 types:
  - 0 = INDIVIDUAL_ADDR (per-vpid, per-addr).
  - 1 = SINGLE_CONTEXT (per-vpid all-addr).
  - 2 = ALL_CONTEXTS (all-vpids; rarely used).
  - 3 = SINGLE_CONTEXT_RETAIN_GLOBALS (per-vpid except global).

REQ-7: TLB-flush hooks:
- `vmx_flush_tlb_current(vcpu)`: invvpid(SINGLE_CONTEXT, vcpu.vpid).
- `vmx_flush_tlb_guest(vcpu)`: invvpid(SINGLE_CONTEXT, vcpu.vpid) — for guest-mode flush.
- `vmx_flush_tlb_user(vcpu)`: invvpid(SINGLE_CONTEXT_RETAIN_GLOBALS, vcpu.vpid).
- `vmx_flush_tlb_all(vcpu)`: invvpid(ALL_CONTEXTS).

REQ-8: Guest CR3 change:
- KVM's `vmx_load_mmu_pgd(vcpu, root_hpa)` programs vmcs.guest_cr3 + invvpid(SINGLE_CONTEXT, vcpu.vpid).
- Eliminates full-TLB-flush on per-process context-switch within guest.

REQ-9: Cross-CPU vCPU migrate:
- When vCPU migrates to different host CPU: per-spec must flush stale TLB.
- KVM uses invvpid(SINGLE_CONTEXT) to ensure no stale TLB on new CPU.

REQ-10: Nested-VMX VPID:
- L1 may use VPID for L2.
- L0 assigns DISTINCT VPID to L2 (vmcs02.vpid != vmcs01.vpid).
- Avoids TLB-aliasing across L1/L2.

REQ-11: VPID 0 reserved:
- Used for "no-VPID-tagging" mode (legacy compat).
- Per-Intel-spec, VPID 0 must NEVER be assigned to a vCPU.

REQ-12: VMXOFF + VPID flush:
- vpid_sync_global at VMX-disable.
- Avoid stale TLB if VMX re-enabled with reused VPIDs.

## Acceptance Criteria

- [ ] AC-1: KVM-VMX module loaded with enable_vpid=1: per-vCPU vmcs.virtual_processor_id != 0.
- [ ] AC-2: Cross-VM context-switch: TLB-miss-rate substantially reduced vs enable_vpid=0 baseline.
- [ ] AC-3: Guest CR3-change: only per-vpid invvpid issued; not full TLB-flush.
- [ ] AC-4: Multi-VM stress: 16 VMs × 4 vCPUs each = 64 vpids assigned; bitmap correctly tracks.
- [ ] AC-5: VM-destroy: VPIDs returned to pool; subsequent VM allocates from same pool.
- [ ] AC-6: Cross-CPU migrate: vCPU moves CPU; INVVPID-flush eliminates stale TLB; no functional regression.
- [ ] AC-7: Nested-VMX: L1 with vmcs01.vpid=N; L2 with vmcs02.vpid=M ≠ N.
- [ ] AC-8: VMX-disable: vpid_sync_global runs; subsequent VMX-enable reuses pool cleanly.
- [ ] AC-9: kvm-unit-tests `vpid` test passes.
- [ ] AC-10: VPID exhaustion: > 16383 concurrent vCPUs degrades to no-tagging mode (vpid=0).

## Architecture

`Vmx` static state:

```
struct VmxStatic {
  vpid_bitmap: KBox<[u64; VMX_NR_VPIDS / 64]>,
  vpid_lock: SpinLock<()>,
  enable_vpid: bool,
}
```

Per-vCPU additions (already in VcpuVmx; see x86-vmx.md):

```
struct VcpuVmx {
  ...
  vpid: u16,
}
```

`Vmx::allocate_vpid()`:
1. If !enable_vpid: return 0.
2. Acquire VPID_LOCK.
3. vpid := find_first_zero_bit(&VPID_BITMAP, VMX_NR_VPIDS).
4. If vpid == VMX_NR_VPIDS: vpid = 0; goto unlock.
5. set_bit(vpid, &VPID_BITMAP).
6. unlock.
7. Return vpid.

`Vmx::vpid_sync_context(vpid)`:
1. If !enable_vpid OR !cpu_has_vmx_invvpid_single():
   - Fall back to vmx_flush_tlb_global.
2. Else: invvpid(SINGLE_CONTEXT, &operand).

`Vmx::vpid_sync_vcpu_addr(vpid, addr)`:
1. If cpu_has_vmx_invvpid_individual_addr():
   - invvpid(INDIVIDUAL_ADDR, &operand{ vpid, addr }).
2. Else:
   - invvpid(SINGLE_CONTEXT, &operand{ vpid }).

`VcpuVmx::create(vcpu)` (extended for VPID):
1. ...[other init]...
2. vcpu.vpid := Vmx::allocate_vpid().
3. vmcs.virtual_processor_id = vcpu.vpid.
4. If vcpu.vpid != 0: vmcs.secondary_exec_control |= ENABLE_VPID.
5. Vmx::vpid_sync_context(vcpu.vpid) (eliminate any pre-existing TLB for this vpid).

`VmxNested::assign_vpid(vmx, &vmcs02)`:
1. nested_vpid := Vmx::allocate_vpid() (fresh from pool).
2. vmcs02.virtual_processor_id = nested_vpid.
3. Vmx::vpid_sync_context(nested_vpid).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vpid_bitmap_no_oob` | OOB | per-VPID alloc bounded by VMX_NR_VPIDS = 16384. |
| `vpid_zero_reserved` | INVARIANT | VPID 0 never assigned to active vCPU. |
| `vpid_unique_per_vcpu` | INVARIANT | per-vCPU vpid distinct across all active vCPUs concurrently. |
| `vpid_release_paired_with_alloc` | INVARIANT | every alloc paired with eventual release on vCPU destroy. |
| `invvpid_args_validated` | INVARIANT | invvpid type ∈ {0..3}; defense against malformed instruction. |

### Layer 2: TLA+

`virt/kvm/vpid_alloc.tla`:
- Per-VPID state ∈ {Free, Assigned(vcpu), Released}.
- Properties:
  - `safety_unique_assignment` — per-VPID at most one Assigned state at a time.
  - `safety_release_after_destroy` — Assigned → Released on vCPU destroy.
  - `liveness_eventual_release` — every Assigned eventually Released or VM-shutdown.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vmx::allocate_vpid` post: returned vpid bit-set in bitmap; or 0 if pool exhausted | `Vmx::allocate_vpid` |
| `Vmx::free_vpid` post: vpid bit-clear in bitmap | `Vmx::free_vpid` |
| `VcpuVmx::create` post: vcpu.vpid assigned; vmcs.virtual_processor_id matches | `VcpuVmx::create` |
| `Vmx::vpid_sync_context` post: per-vpid TLB entries invalidated | `Vmx::vpid_sync_context` |
| Per-host VPID-bitmap consistent (alloc/free balanced) | invariants on alloc/free pair |

### Layer 4: Verus/Creusot functional

`Per-vCPU VPID assignment + per-CR3-change INVVPID = no TLB-leak across vCPUs` semantic equivalence: per-vCPU TLB entries visible only when vCPU-running with matching VPID; cross-vCPU access blocked by VPID-tagging.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

VPID-specific reinforcement:

- **Per-vCPU VPID distinct** — defense against TLB-leak across vCPUs.
- **VPID 0 reserved** — defense against per-spec violation causing CPU exception.
- **VPID bitmap protected by spinlock** — defense against concurrent alloc race.
- **VPID release on vCPU destroy** — defense against bitmap-leak.
- **INVVPID after every cross-CPU vCPU migrate** — defense against stale TLB on new CPU.
- **Nested-VPID distinct from L1-VPID** — defense against L1/L2 TLB-aliasing.
- **VPID exhaustion fallback to vpid=0** — defense against allocation-failure crashing VM.
- **VMXOFF flush** — defense against post-disable stale TLB.
- **Per-spec invvpid-type whitelist** — defense against fluid type causing CPU #UD.
- **vmcs.secondary_exec_control bit-validate** — defense against guest manipulating ENABLE_VPID bit.
- **Per-vCPU VPID accounting** — defense against ID-leak via vCPU-rapid-create-destroy attack.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VMX vendor (covered in `x86-vmx.md` Tier-3)
- TDP MMU (covered in `x86-mmu-tdp.md` Tier-3; related but separate feature)
- KVM core (covered in `kvm-core.md` Tier-3)
- AMD ASID (covered in `x86-svm.md` Tier-3; AMD analog of VPID)
- Implementation code
