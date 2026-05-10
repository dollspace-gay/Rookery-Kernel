# Tier-3: arch/x86/kvm/vmx/vmx.c (MSR autoload subset) — VMX MSR auto-load/store list + MSR pass-through bitmap (per-vCPU per-MSR access policy)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-vmx.md
upstream-paths:
  - arch/x86/kvm/vmx/vmx.c (MSR autoload subset; see grep "msr_autoload|VM_ENTRY_MSR_LOAD|VM_EXIT_MSR")
  - arch/x86/kvm/vmx/vmx.h (msr_autoload struct)
  - arch/x86/kvm/x86.c (kvm_set_msr_filter for general MSR policy)
-->

## Summary

KVM VMX provides two complementary mechanisms for MSR virtualization: (a) MSR auto-load/store lists let CPU automatically save/restore selected guest MSRs at vmenter/vmexit (vmcs.VM_ENTRY_MSR_LOAD_ADDR + COUNT for entry; VM_EXIT_MSR_STORE/LOAD for exit) — typically per-vCPU IA32_EFER, IA32_PERF_GLOBAL_CTRL, IA32_FS/GS_BASE; (b) MSR pass-through bitmap (vmcs.MSR_BITMAP_ADDR) is per-vCPU 4KiB bitmap with 4 sub-bitmaps (low-MSR R, low-MSR W, high-MSR R, high-MSR W) — bit-set = vmexit-on-access, bit-clear = direct guest access. KVM dynamically toggles bitmap bits to optimize hot-path MSR access (e.g., when LBR active, allow guest direct LBR-MSR access).

This Tier-3 covers MSR autoload + bitmap code in `vmx.c` (~300-400 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct msr_autoload` | per-vCPU autoload list | `kernel::kvm::x86::vmx::MsrAutoload` |
| `struct vmx_msrs` | per-vCPU per-direction MSR list | `VmxMsrs` |
| `vmx_add_atomic_switch_msr(vmx, msr, guest_val, host_val, entry_only)` | add MSR to autoload | `VcpuVmx::add_atomic_switch_msr` |
| `vmx_clear_atomic_switch_msr(vmx, msr)` | remove MSR from autoload | `VcpuVmx::clear_atomic_switch_msr` |
| `vmx_find_loadstore_msr_slot(...)` | per-MSR-idx slot lookup | `VcpuVmx::find_loadstore_msr_slot` |
| `add_atomic_switch_msr(vmx, msr, guest_val, host_val, entry_only)` | helper | `VcpuVmx::add_atomic_switch_msr` |
| `vmx_remove_auto_msr(...)` | remove from per-direction list | `VcpuVmx::remove_auto_msr` |
| `vmx_add_auto_msr(...)` | add to per-direction list | `VcpuVmx::add_auto_msr` |
| `vmx_set_intercept_for_msr(vcpu, msr, type, value)` | per-MSR-bitmap toggle | `VcpuVmx::set_intercept_for_msr` |
| `vmx_disable_intercept_for_msr(...)` / `_enable_intercept_for_msr(...)` | bitmap-bit-clear / bit-set | `VcpuVmx::disable_intercept_for_msr` / `_enable_intercept_for_msr` |
| `vmx_msr_bitmap_l01_changed(...)` | per-vmexit bitmap-changed marker | `VcpuVmx::msr_bitmap_l01_changed` |
| `init_vmcs_msr_bitmap(vmx)` | per-vCPU initial bitmap | `VcpuVmx::init_msr_bitmap` |
| `update_msr_bitmap_x2apic(...)` | per-mode x2APIC bitmap update | `VcpuVmx::update_msr_bitmap_x2apic` |
| `kvm_msr_filter` (KVM_X86_SET_MSR_FILTER) | per-VM MSR access policy | covered in `x86-msr-filter.md` |
| `VM_ENTRY_MSR_LOAD_ADDR` / `COUNT` | per-vmenter auto-load list | UAPI |
| `VM_EXIT_MSR_LOAD_ADDR` / `COUNT` | per-vmexit auto-load (host) list | UAPI |
| `VM_EXIT_MSR_STORE_ADDR` / `COUNT` | per-vmexit auto-store (guest) list | UAPI |
| `MSR_BITMAP_ADDR` | per-vCPU MSR bitmap pointer | UAPI |
| `vmx_setup_uret_msr(...)` | URET (User-Return) MSR list (older mechanism) | `VcpuVmx::setup_uret_msr` |

## Compatibility contract

REQ-1: Per-vCPU `msr_autoload`:
- `host` (VmxMsrs; per-CPU host values to restore on vmexit).
- `guest` (VmxMsrs; per-vCPU guest values to load on vmenter).
- Each `vmx_msrs`: { val[NR_AUTOLOAD_MSRS], nr } (typically 8 slots).

REQ-2: Per-MSR-entry layout:
- `index` (u32 MSR index).
- `reserved` (u32; padding).
- `value` (u64).

REQ-3: Per-vmenter MSR auto-load:
- vmcs.VM_ENTRY_MSR_LOAD_ADDR = phys-addr of msr_autoload.guest.val.
- vmcs.VM_ENTRY_MSR_LOAD_COUNT = msr_autoload.guest.nr.
- CPU reads count entries; for each: WRMSR(index, value).
- Used for MSRs needing per-vCPU specific value (e.g., guest IA32_EFER vs host IA32_EFER).

REQ-4: Per-vmexit MSR auto-store:
- vmcs.VM_EXIT_MSR_STORE_ADDR = phys-addr of guest val.
- vmcs.VM_EXIT_MSR_STORE_COUNT = nr.
- CPU writes current MSR values back to entries.
- Used when guest may have modified MSRs that KVM needs to read post-vmexit.

REQ-5: Per-vmexit MSR auto-load (host):
- vmcs.VM_EXIT_MSR_LOAD_ADDR = phys-addr of host val.
- vmcs.VM_EXIT_MSR_LOAD_COUNT = nr.
- CPU restores host values on vmexit.

REQ-6: MSR pass-through bitmap (4KiB):
- Layout per Intel SDM Vol 3:
  - Bytes 0..0x3FF (1KiB): low-MSR (0..0x1FFF) READ intercept; bit-set = intercept.
  - Bytes 0x400..0x7FF (1KiB): high-MSR (0xC0000000..0xC0001FFF) READ intercept.
  - Bytes 0x800..0xBFF (1KiB): low-MSR WRITE intercept.
  - Bytes 0xC00..0xFFF (1KiB): high-MSR WRITE intercept.
- Bit-clear: guest MSR access without vmexit.
- Bit-set: vmexit on guest MSR access; KVM emulates.

REQ-7: Per-vCPU MSR_BITMAP allocation:
- Per-vCPU 4KiB page allocated at create.
- Initially all-set (all-intercept).
- KVM dynamically clears bits for high-traffic MSRs (allow direct access).

REQ-8: Common pass-through MSRs:
- IA32_FS_BASE / IA32_GS_BASE: per-thread, used by syscall.
- IA32_KERNEL_GS_BASE: per-thread, used by syscall.
- IA32_TSC_AUX: per-CPU TSC auxiliary; rdtscp.
- IA32_SPEC_CTRL: per-thread Spectre-v2 mitigation (always pass-through if available).
- LBR_FROM/_TO/_INFO/_TOS: per-LBR-active (covered in x86-vmx-lbr.md).
- PEBS-related: per-PEBS-active (covered in x86-vmx-pebs.md).

REQ-9: Always-intercept MSRs:
- IA32_DEBUGCTLMSR (LBR-control flag; KVM toggles LBR pass-through based on this).
- IA32_BNDCFGS (MPX; deprecated).
- IA32_PERF_GLOBAL_CTRL (auto-load instead).
- Per-VM secure-boot related MSRs.

REQ-10: KVM_SET_MSRS / KVM_GET_MSRS for migration:
- Per-vCPU MSRs migrated.
- KVM's "msrs_to_save" + "emulated_msrs" lists determine what migrates.

REQ-11: Per-MSR access nuances:
- Some MSRs require both autoload AND emulation (e.g., IA32_PERF_GLOBAL_CTRL: autoload for active counters; emulate for control updates that affect non-active counters).
- KVM tracks per-vCPU per-MSR state in shadows.

REQ-12: Nested-VMX MSR bitmap:
- L0 + L1 bitmaps must be ORed for L2.
- KVM maintains separate bitmaps for L1 (msr_bitmap_l01) and L2 (msr_bitmap_nested_l1).

## Acceptance Criteria

- [ ] AC-1: Per-vCPU MSR_BITMAP allocated at vCPU create; vmcs.MSR_BITMAP_ADDR set.
- [ ] AC-2: Pass-through IA32_FS_BASE: guest WRMSR(0xC0000100, X) → no vmexit (direct access).
- [ ] AC-3: Intercepted IA32_DEBUGCTL: guest WRMSR(0x1D9, X) → vmexit; KVM emulates.
- [ ] AC-4: Auto-load IA32_EFER: vmcs.VM_ENTRY_MSR_LOAD includes EFER; per-vmenter loaded; per-vmexit restored to host.
- [ ] AC-5: LBR pass-through dynamic toggle: enable LBR via DEBUGCTL → bitmap clears LBR-MSR bits → next read direct; disable → bitmap sets bits.
- [ ] AC-6: x2APIC mode: bitmap updated for x2APIC MSR range (0x800-0x83F) per virtualize-x2APIC-mode policy.
- [ ] AC-7: Live migration: per-vCPU MSR shadows preserved; post-migrate guest reads consistent.
- [ ] AC-8: Per-MSR autoload slot bound (8 typical): adding > 8 MSRs returns -ENOSPC.
- [ ] AC-9: kvm-unit-tests `msr_pass_through` test passes.
- [ ] AC-10: Performance: per-pass-through MSR access < 50 cycles (no vmexit) vs ~2000 cycles per vmexit-emulation.

## Architecture

`MsrAutoload` per-vCPU (already in VcpuVmx; see x86-vmx.md):

```
struct MsrAutoload {
  host: VmxMsrs,
  guest: VmxMsrs,
}

struct VmxMsrs {
  val: KBox<[VmxMsrEntry; NR_AUTOLOAD_MSRS]>,
  nr: u32,
}

#[repr(C)]
struct VmxMsrEntry {
  index: u32,
  reserved: u32,
  value: u64,
}
```

Per-vCPU MSR bitmap (already in VcpuVmx):

```
struct VcpuVmx {
  ...
  msr_bitmap: KBox<[u8; 4096]>,                 // 4KiB
  msr_bitmap_l01: u64,                          // physical addr of bitmap
}
```

`VcpuVmx::add_atomic_switch_msr(vmx, msr, guest_val, host_val, entry_only)`:
1. m := &vmx.msr_autoload.
2. slot := find_loadstore_msr_slot(&m.guest, msr).
3. If slot < 0: alloc new slot.
4. m.guest.val[slot] = { index: msr, value: guest_val }.
5. If !entry_only: m.host.val[slot] = { index: msr, value: host_val }.
6. m.guest.nr = max(m.guest.nr, slot + 1).
7. Update vmcs.VM_ENTRY_MSR_LOAD_COUNT = m.guest.nr.

`VcpuVmx::set_intercept_for_msr(vcpu, msr, type, intercept)`:
1. bitmap := vmx.msr_bitmap.
2. If msr in low-range (0..0x1FFF):
   - READ: bit_idx = msr.
   - WRITE: bit_idx = 0x4000 + msr.
3. Else if msr in high-range (0xC0000000..0xC0001FFF):
   - READ: bit_idx = 0x2000 + (msr - 0xC0000000).
   - WRITE: bit_idx = 0x6000 + (msr - 0xC0000000).
4. Else: panic (invalid MSR range).
5. If intercept: set_bit(bit_idx, bitmap).
6. Else: clear_bit(bit_idx, bitmap).

`VcpuVmx::init_msr_bitmap(vmx)`:
1. Allocate 4KiB bitmap; init all-set (intercept everything).
2. For each "always-pass-through" MSR: clear bits.
3. vmcs.MSR_BITMAP_ADDR = bitmap_pa.
4. vmcs.exec_control |= USE_MSR_BITMAPS.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `msr_autoload_slot_bounded` | OOB | per-vCPU autoload slot < NR_AUTOLOAD_MSRS. |
| `msr_bitmap_aligned` | INVARIANT | per-vCPU bitmap 4KiB-aligned; defense against MSR_BITMAP_ADDR mis-alignment. |
| `msr_idx_bitmap_match` | INVARIANT | per-MSR bit-idx in correct 1KiB sub-bitmap based on range. |
| `bitmap_state_consistent` | INVARIANT | per-MSR intercept state matches per-vCPU policy (LBR/PEBS/x2APIC). |
| `pass_through_only_safe_msrs` | INVARIANT | only architecturally-safe MSRs default to pass-through. |

### Layer 2: TLA+

`virt/kvm/msr_pass_through.tla`:
- Per-vCPU per-MSR access policy: {Intercepted, PassThroughR, PassThroughW, PassThroughRW}.
- Properties:
  - `safety_intercepted_implies_vmexit` — Intercepted MSR access generates vmexit.
  - `safety_pass_through_implies_no_vmexit` — PassThrough access goes direct.
  - `safety_dynamic_toggle_atomic` — bitmap update visible atomically per-vCPU.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuVmx::add_atomic_switch_msr` post: slot populated; nr updated; vmcs count refreshed | `VcpuVmx::add_atomic_switch_msr` |
| `VcpuVmx::clear_atomic_switch_msr` post: slot freed; nr decremented | `VcpuVmx::clear_atomic_switch_msr` |
| `VcpuVmx::set_intercept_for_msr` post: bitmap-bit reflects desired state | `VcpuVmx::set_intercept_for_msr` |
| Per-vCPU autoload list synchronized with vmcs counts | invariants on autoload changes |

### Layer 4: Verus/Creusot functional

`Per-vmenter: guest sees per-MSR auto-load values; per-vmexit: host MSRs restored to pre-vmenter values` semantic equivalence: per-MSR round-trip preserves both per-vCPU and host state.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

MSR-autoload-specific reinforcement:

- **Per-vCPU autoload slot cap (NR_AUTOLOAD_MSRS=8 typical)** — defense against unbounded list causing per-vmenter overhead.
- **Per-vCPU bitmap 4KiB-aligned** — defense against MSR_BITMAP_ADDR mis-alignment causing vmexit-on-access of all MSRs.
- **Default-intercept policy** — defense against accidental MSR-leak via uninitialized bitmap.
- **Per-MSR safety classification** — only architecturally-safe MSRs default to pass-through.
- **Per-LBR/PEBS/x2APIC dynamic toggle under apic.lock** — defense against torn intercept-state.
- **Per-vmexit autoload count refresh** — defense against stale count after add/remove race.
- **Per-VM KVM_X86_SET_MSR_FILTER orthogonal** — defense against bitmap-allowing-but-policy-denying combinations.
- **Nested-VMX bitmap merge correct** — defense against L1-leak bypassing L0 intercept.
- **Per-vCPU IA32_SPEC_CTRL pass-through coordinated with retpoline** — defense against Spectre-v2.
- **Per-VM autoload guest-list distinct from host-list** — defense against value-leak.
- **Per-vCPU autoload clears on vCPU destroy** — defense against stale autoload referencing freed memory.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VMX core (covered in `x86-vmx.md` Tier-3)
- KVM_X86_SET_MSR_FILTER (covered in `x86-msr-filter.md` Tier-3)
- LBR pass-through specifics (covered in `x86-vmx-lbr.md` Tier-3)
- PEBS pass-through (covered in `x86-vmx-pebs.md` Tier-3)
- x2APIC bitmap (covered in `x86-x2apic.md` Tier-3)
- AMD MSR pass-through (covered in `x86-svm.md` Tier-3; analog mechanism)
- Implementation code
