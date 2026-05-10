# Tier-3: arch/x86/kvm/vmx/vmx.c (msr-autoload subset) — VMX per-vCPU automatic MSR-load/store on vmenter/vmexit

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-vmx.md
upstream-paths:
  - arch/x86/kvm/vmx/vmx.c (msr-autoload subset: ~1023..1180, ~4893..4900, ~6529)
  - arch/x86/include/asm/vmx.h (VMX_MSR_LOAD_LIST_SIZE / vmx_msr_entry)
-->

## Summary

VMX vmcs.{vm-entry,vm-exit}-msr-{load,store}-list provides hardware-managed atomic MSR save/restore around vmenter/vmexit, avoiding manual RDMSR/WRMSR-per-MSR overhead. Per-vCPU has 3 lists: **guest-load** (loaded on vmenter), **host-load** (loaded on vmexit), **autostore** (saved on vmexit from guest before host-load). Per-list is array of `vmx_msr_entry { index, reserved, value }`. Per-vCPU `add_atomic_switch_msr(msr, guest_val, host_val)` registers atomic-switch (e.g. MSR_EFER, MSR_IA32_PERF_GLOBAL_CTRL). Per-special-case (EFER, PERF_GLOBAL_CTRL) HW directly handles via dedicated vmcs fields. Critical for: per-vmenter/vmexit performance — atomic MSR swapping without explicit instructions.

This Tier-3 covers msr-autoload subset of `vmx.c` (~200 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct msr_autoload` | per-vCPU list of (guest, host) | `MsrAutoload` |
| `struct vmx_msrs` | per-list (val + nr) | `VmxMsrs` |
| `struct vmx_msr_entry` | per-entry { index, reserved, value } | `VmxMsrEntry` |
| `vmx->msr_autoload` | per-vCPU autoload state | `VmxVcpu::msr_autoload` |
| `vmx->msr_autostore` | per-vCPU autostore state | `VmxVcpu::msr_autostore` |
| `add_atomic_switch_msr()` | per-vCPU register MSR for autoload | `Vmx::add_atomic_switch_msr` |
| `clear_atomic_switch_msr()` | per-vCPU unregister | `Vmx::clear_atomic_switch_msr` |
| `add_atomic_switch_msr_special()` | per-special MSR via vmcs field | `Vmx::add_atomic_switch_msr_special` |
| `vmx_add_auto_msr()` | per-list add | `Vmx::add_auto_msr` |
| `vmx_remove_auto_msr()` | per-list remove | `Vmx::remove_auto_msr` |
| `vmx_find_loadstore_msr_slot()` | per-list slot lookup | `Vmx::find_loadstore_msr_slot` |
| `add_atomic_store_msr()` | per-vCPU autostore add | `Vmx::add_atomic_store_msr` |
| `VM_ENTRY_MSR_LOAD_COUNT` (vmcs field) | per-list count (entry-load) | UAPI |
| `VM_EXIT_MSR_LOAD_COUNT` (vmcs field) | per-list count (exit-load) | UAPI |
| `VM_EXIT_MSR_STORE_COUNT` (vmcs field) | per-list count (autostore) | UAPI |
| `VM_ENTRY_MSR_LOAD_ADDR` / `VM_EXIT_MSR_LOAD_ADDR` / `VM_EXIT_MSR_STORE_ADDR` | per-list phys-addr | UAPI |
| `VMX_MSR_LIST_SIZE` (max ~512) | per-list cap | shared |

## Compatibility contract

REQ-1: Per-list layout:
- struct vmx_msr_entry { u32 index, u32 reserved, u64 value }; // 16 bytes per entry.
- Per-list: array of entries + nr count.
- Per-vmcs field stores phys-addr + count.

REQ-2: Per-vCPU 3 lists:
- msr_autoload.guest: loaded on vmenter (via VM_ENTRY_MSR_LOAD_*).
- msr_autoload.host: loaded on vmexit (via VM_EXIT_MSR_LOAD_*).
- msr_autostore: stored on vmexit (via VM_EXIT_MSR_STORE_*).

REQ-3: add_atomic_switch_msr(vmx, msr, guest_val, host_val):
- Per-special-case (EFER, PERF_GLOBAL_CTRL): use vmcs dedicated field, no list.
- Else:
  - vmx_add_auto_msr(&autoload.guest, msr, guest_val, VM_ENTRY_MSR_LOAD_COUNT).
  - vmx_add_auto_msr(&autoload.host, msr, host_val, VM_EXIT_MSR_LOAD_COUNT).

REQ-4: vmx_add_auto_msr(list, msr, val, count_field):
- slot = vmx_find_loadstore_msr_slot(list, msr).
- If slot found: list.val[slot].value = val.
- Else: alloc new slot at list.nr; list.val[list.nr] = {msr, 0, val}; list.nr++.
- vmcs_write32(count_field, list.nr).

REQ-5: clear_atomic_switch_msr(vmx, msr):
- vmx_remove_auto_msr(&autoload.guest, msr, VM_ENTRY_MSR_LOAD_COUNT).
- vmx_remove_auto_msr(&autoload.host, msr, VM_EXIT_MSR_LOAD_COUNT).

REQ-6: vmx_remove_auto_msr(list, msr, count_field):
- slot = vmx_find_loadstore_msr_slot(list, msr).
- If slot found: memmove list.val[slot..nr-1] = list.val[slot+1..nr]; list.nr--.
- vmcs_write32(count_field, list.nr).

REQ-7: Per-special-case MSR_EFER:
- Use vmcs.GUEST_IA32_EFER + VM_ENTRY_LOAD_IA32_EFER (entry-control bit) + vmcs.HOST_IA32_EFER + VM_EXIT_LOAD_IA32_EFER (exit-control bit).
- HW handles atomically.

REQ-8: Per-special-case MSR_IA32_PERF_GLOBAL_CTRL:
- Same pattern: vmcs.GUEST_IA32_PERF_GLOBAL_CTRL + entry-control bit.

REQ-9: Per-autostore (vmexit-side):
- Per-vCPU: when guest writes MSR via WRMSR (intercepted), KVM updates shadow + autostore.
- On vmexit: HW writes MSRs in autostore-list to memory, KVM reads back guest's last value.

REQ-10: Per-VMCS init:
- vmcs_write64(VM_ENTRY_MSR_LOAD_ADDR, phys(autoload.guest.val)).
- vmcs_write64(VM_EXIT_MSR_LOAD_ADDR, phys(autoload.host.val)).
- vmcs_write64(VM_EXIT_MSR_STORE_ADDR, phys(autostore.val)).
- vmcs_write32(*_COUNT, 0).

REQ-11: Per-list size limit:
- VMX_MSR_LIST_SIZE typically 512 (per-vmcs HW limit).

## Acceptance Criteria

- [ ] AC-1: vmcs_init: VM_ENTRY/EXIT_MSR_LOAD_ADDR set; counts = 0.
- [ ] AC-2: add_atomic_switch_msr(MSR_EFER, ...): special-case handled via vmcs.GUEST_IA32_EFER.
- [ ] AC-3: add_atomic_switch_msr(MSR_IA32_KERNEL_GS_BASE, ...): added to autoload guest+host lists.
- [ ] AC-4: clear_atomic_switch_msr(MSR_IA32_KERNEL_GS_BASE): removed from lists.
- [ ] AC-5: VM_ENTRY_MSR_LOAD_COUNT updated on add/remove.
- [ ] AC-6: Per-vmenter: HW loads guest-list MSRs.
- [ ] AC-7: Per-vmexit: HW saves autostore MSRs + loads host-list MSRs.
- [ ] AC-8: List size > VMX_MSR_LIST_SIZE: -ENOSPC.
- [ ] AC-9: vmx_find_loadstore_msr_slot returns existing slot for known MSR.
- [ ] AC-10: kvm-unit-tests `msr_load_store` test passes.

## Architecture

Per-vCPU state:

```
struct VmxMsrs {
  val: Box<[VmxMsrEntry; VMX_MSR_LIST_SIZE]>,    // page-sized
  nr: u32,
}

#[repr(C)]
struct VmxMsrEntry {
  index: u32,
  reserved: u32,
  value: u64,
}

struct MsrAutoload {
  guest: VmxMsrs,                                 // VM_ENTRY_MSR_LOAD
  host: VmxMsrs,                                  // VM_EXIT_MSR_LOAD
}

struct VmxVcpu {
  ...
  msr_autoload: MsrAutoload,
  msr_autostore: VmxMsrs,                         // VM_EXIT_MSR_STORE
}

const VMX_MSR_LIST_SIZE: usize = 512;
```

`Vmx::find_loadstore_msr_slot(m, msr) -> Option<usize>`:
1. for i in 0..m.nr: if m.val[i].index == msr: return Some(i).
2. None.

`Vmx::add_auto_msr(m, msr, val, count_field, kvm) -> Result<()>`:
1. slot = Vmx::find_loadstore_msr_slot(m, msr).
2. if let Some(idx) = slot:
   - m.val[idx].value = val.
   - return Ok.
3. else:
   - if m.nr == VMX_MSR_LIST_SIZE: return Err(NoSpace).
   - m.val[m.nr] = VmxMsrEntry { index: msr, reserved: 0, value: val }.
   - m.nr++.
   - vmcs_write32(count_field, m.nr).
4. Ok.

`Vmx::remove_auto_msr(m, msr, count_field)`:
1. slot = Vmx::find_loadstore_msr_slot(m, msr).
2. if let Some(idx) = slot:
   - for i in idx..m.nr-1: m.val[i] = m.val[i+1].
   - m.nr--.
   - vmcs_write32(count_field, m.nr).

`Vmx::add_atomic_switch_msr(vmx, msr, guest_val, host_val, kvm) -> Result<()>`:
1. m = &vmx.msr_autoload.
2. /* Special-case */
3. match msr:
   - MSR_EFER: Vmx::add_atomic_switch_msr_special(vmx, VM_ENTRY_LOAD_IA32_EFER, VM_EXIT_LOAD_IA32_EFER, GUEST_IA32_EFER, HOST_IA32_EFER, guest_val, host_val); return.
   - MSR_IA32_PERF_GLOBAL_CTRL: similar with PERF-GLOBAL-CTRL fields; return.
4. /* Generic case */
5. err = Vmx::add_auto_msr(&m.guest, msr, guest_val, VM_ENTRY_MSR_LOAD_COUNT, kvm)?
6. err = Vmx::add_auto_msr(&m.host, msr, host_val, VM_EXIT_MSR_LOAD_COUNT, kvm)?

`Vmx::clear_atomic_switch_msr(vmx, msr)`:
1. m = &vmx.msr_autoload.
2. /* Special-case clear via vmcs control bits */
3. Vmx::remove_auto_msr(&m.guest, msr, VM_ENTRY_MSR_LOAD_COUNT).
4. Vmx::remove_auto_msr(&m.host, msr, VM_EXIT_MSR_LOAD_COUNT).

`Vmx::add_atomic_store_msr(vmx, msr, kvm) -> Result<()>`:
1. err = Vmx::add_auto_msr(&vmx.msr_autostore, msr, 0, VM_EXIT_MSR_STORE_COUNT, kvm)?
2. Ok.

`Vmx::add_atomic_switch_msr_special(vmx, entry_control, exit_control, guest_field, host_field, guest_val, host_val)`:
1. vmcs_set_bits(VM_ENTRY_CONTROLS, entry_control).
2. vmcs_set_bits(VM_EXIT_CONTROLS, exit_control).
3. vmcs_writeXX(guest_field, guest_val).
4. vmcs_writeXX(host_field, host_val).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nr_le_list_size` | INVARIANT | per-list.nr ≤ VMX_MSR_LIST_SIZE. |
| `count_field_eq_nr` | INVARIANT | post-add/remove: vmcs.*_COUNT == m.nr. |
| `entry_index_unique_per_list` | INVARIANT | per-list: at most one entry per msr. |
| `special_via_vmcs_field` | INVARIANT | EFER / PERF_GLOBAL_CTRL handled via vmcs.*_IA32_* fields. |
| `addr_phys_aligned` | INVARIANT | per-LOAD/STORE_ADDR 16-byte aligned. |

### Layer 2: TLA+

`virt/kvm/vmx_msr_autoload.tla`:
- Per-vCPU autoload list state + per-vmenter/vmexit HW load/store.
- Properties:
  - `safety_no_overlap_special_and_generic` — per-special-MSR not in generic list.
  - `safety_post_vmexit_host_msr` — post-vmexit: host MSRs match host-list values.
  - `safety_post_vmenter_guest_msr` — post-vmenter: guest MSRs match guest-list values.
  - `liveness_clear_after_vCPU_destroy` — per-vCPU destroy: lists freed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vmx::add_auto_msr` post: msr in list with given value; nr incremented if new | `Vmx::add_auto_msr` |
| `Vmx::remove_auto_msr` post: msr removed; nr decremented | `Vmx::remove_auto_msr` |
| `Vmx::add_atomic_switch_msr` post: per-MSR registered (special or generic) | `Vmx::add_atomic_switch_msr` |
| `Vmx::clear_atomic_switch_msr` post: per-MSR removed | `Vmx::clear_atomic_switch_msr` |

### Layer 4: Verus/Creusot functional

`Per-vmenter HW loads guest MSRs from VM_ENTRY_MSR_LOAD list; per-vmexit HW saves guest MSRs to VM_EXIT_MSR_STORE list + loads host MSRs from VM_EXIT_MSR_LOAD list` semantic equivalence: per-Intel SDM §27.4 + §27.5 (VMX MSR Load/Store).

## Hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

MSR-autoload-specific reinforcement:

- **Per-list nr bounded by VMX_MSR_LIST_SIZE** — defense against per-list HW limit overrun.
- **Per-MSR uniqueness in list** — defense against per-MSR conflict.
- **Per-special-case via vmcs field** — defense against per-list slot consumption when HW-supports field.
- **Per-list page locked in host phys-addr** — defense against host swap moving page mid-vmenter.
- **Per-vmcs.*_COUNT consistent with list.nr** — defense against per-vmenter HW reading garbage entries.
- **Per-add fails gracefully on full list** — defense against silent overflow.
- **Per-remove memmove maintains contiguity** — defense against per-slot leak.
- **Per-vCPU destroy frees lists** — defense against per-VM-leak.
- **Per-add_atomic_switch_msr serialized via vmx_load_lock** — defense against concurrent-vCPU race.
- **Per-special-case validated against feature MSRs** — defense against per-non-supported MSR claimed-special.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VMX core (covered in `x86-vmx.md` Tier-3)
- VMX VMENTER trampoline (covered in `x86-vmx-vmenter.md` Tier-3)
- VMX MSR-bitmap intercept (covered in `x86-msr-passthrough.md` Tier-3)
- KVM MSR-filter (covered in `x86-msr-filter.md` Tier-3)
- Implementation code
