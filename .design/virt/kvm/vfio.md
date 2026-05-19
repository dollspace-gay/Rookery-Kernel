# Tier-3: virt/kvm/vfio.c — KVM ↔ VFIO integration (assigned-device IOMMU/coherency awareness)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - virt/kvm/vfio.c (~387 lines)
  - virt/kvm/vfio.h
  - drivers/vfio/ (out-of-Tier-3 scope)
-->

## Summary

Per-VM `kvm-vfio` device (registered as KVM_DEV_TYPE_VFIO via KVM_CREATE_DEVICE) maintains the bridge between KVM and userspace VFIO containers/groups holding assigned PCI devices. Per-VM tracks list of `kvm_vfio_file` (bound VFIO container fd or VFIO file fd post-7.x). For each bound file: tracks coherent vs non-coherent DMA capability via `kvm_vfio_file_enforced_coherent`. Per-non-coherent registers via `kvm_arch_register_noncoherent_dma(kvm)` → bumps `kvm.arch.noncoherent_dma_count` → disables PAT-ignore-quirk so EPT memtype follows guest-PAT (see `x86-pat.md`). Critical for: VFIO assigned-device DMA correctness across coherent vs non-coherent buses.

This Tier-3 covers `vfio.c` (~387 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_vfio` | per-VM kvm-vfio state | `KvmVfio` |
| `struct kvm_vfio_file` | per-bound-VFIO-fd entry | `KvmVfioFile` |
| `kvm_vfio_file_set_kvm()` | per-VFIO-fd → kvm bind | `Vfio::file_set_kvm` |
| `kvm_vfio_file_enforced_coherent()` | per-VFIO-fd coherent-cap | `Vfio::file_enforced_coherent` |
| `kvm_vfio_file_is_valid()` | per-VFIO-fd validity check | `Vfio::file_is_valid` |
| `kvm_vfio_file_iommu_group()` | per-VFIO-fd → IOMMU group | `Vfio::file_iommu_group` |
| `kvm_vfio_file_add()` | per-VFIO-fd add to VM | `Vfio::file_add` |
| `kvm_vfio_file_del()` | per-VFIO-fd remove from VM | `Vfio::file_del` |
| `kvm_vfio_set_file()` | per-VFIO-fd attribute setter | `Vfio::set_file` |
| `kvm_vfio_update_coherency()` | per-VM coherency recompute | `Vfio::update_coherency` |
| `kvm_arch_register_noncoherent_dma()` | per-VM bump non-coherent count | `KvmArch::register_noncoherent_dma` |
| `kvm_arch_unregister_noncoherent_dma()` | per-VM dec non-coherent count | `KvmArch::unregister_noncoherent_dma` |
| `KVM_DEV_VFIO_FILE` (replaces _GROUP) | per-attr selector | UAPI |
| `KVM_DEV_VFIO_FILE_ADD` / `_DEL` | per-attr ADD/DEL | UAPI |
| `KVM_DEV_VFIO_FILE_SET_SPAPR_TCE` | per-attr SPAPR | UAPI (ppc-only) |

## Compatibility contract

REQ-1: KVM_CREATE_DEVICE(KVM_DEV_TYPE_VFIO):
- Per-VM allocates `kvm_vfio` state.
- Bound to kvm.devices list.

REQ-2: KVM_DEV_VFIO_FILE_ADD attribute:
- Userspace writes (fd) to KVM_DEV_VFIO_FILE attribute.
- KVM looks up VFIO-fd via fget(fd).
- Validates VFIO-file via kvm_vfio_file_is_valid (checks file.f_op).
- Allocates kvm_vfio_file; binds file → kvm via kvm_vfio_file_set_kvm.
- Updates per-VM coherency.

REQ-3: KVM_DEV_VFIO_FILE_DEL:
- Find kvm_vfio_file by fd; remove from list.
- Unbind via kvm_vfio_file_set_kvm(file, NULL).
- Update per-VM coherency.

REQ-4: Per-coherency tracking:
- Per-VFIO-file may report per-IOMMU-group coherency.
- kvm_vfio_file_enforced_coherent: returns true if VFIO-file enforces DMA-coherent (via vfio_file_enforced_coherent).
- Per-VM scan: noncoherent_dma_count = #{kvf : !file_enforced_coherent(kvf.file)}.
- Per-bump: kvm_arch_register_noncoherent_dma(kvm) → kvm.arch.noncoherent_dma_count++.
- Per-dec: kvm_arch_unregister_noncoherent_dma → count--.

REQ-5: Per-VM PAT-ignore-quirk gating:
- noncoherent_dma_count > 0 → vmx_ignore_guest_pat returns false → EPT memtype = guest-PAT (not WB).
- Critical for assigned-device DMA seeing correct memtype.

REQ-6: KVM_DEV_VFIO_FILE_SET_SPAPR_TCE (ppc-only):
- Per-(VFIO-fd, IOMMU-group, TCE) bind for ppc IOMMU.
- v0 (x86_64): N/A.

REQ-7: Per-VM destroy:
- Iterate kvm_vfio.file_list.
- Per-kvf: kvm_vfio_file_set_kvm(kvf.file, NULL); fput; remove.
- Update coherency to default.

REQ-8: Per-attr ABI:
- KVM_DEV_VFIO_FILE_ADD = 1.
- KVM_DEV_VFIO_FILE_DEL = 2.
- KVM_DEV_VFIO_FILE_SET_SPAPR_TCE = 3 (legacy GROUP path also).

REQ-9: Per-IOMMU-group lookup (ppc-only):
- kvm_vfio_file_iommu_group: per-VFIO-fd resolves to per-iommu_group.

## Acceptance Criteria

- [ ] AC-1: KVM_CREATE_DEVICE(KVM_DEV_TYPE_VFIO): per-VM kvm_vfio allocated.
- [ ] AC-2: KVM_DEV_VFIO_FILE_ADD with VFIO-container fd: kvm_vfio_file added.
- [ ] AC-3: VFIO non-coherent DMA: kvm.arch.noncoherent_dma_count++.
- [ ] AC-4: Per-non-coherent VM: vmx_ignore_guest_pat returns false.
- [ ] AC-5: KVM_DEV_VFIO_FILE_DEL: kvm_vfio_file removed; coherency recomputed.
- [ ] AC-6: VFIO-fd invalid (non-VFIO file): -EINVAL.
- [ ] AC-7: VFIO-fd added twice: -EEXIST.
- [ ] AC-8: VFIO-fd del with not-bound: -ENOENT.
- [ ] AC-9: VM destroy with bound VFIO files: per-file unbound; non-coherent count decremented.
- [ ] AC-10: VFIO-Posted-Interrupts: per-vCPU PiDesc.NDST/NV updated when VFIO IRQ assigned.

## Architecture

Per-VM kvm-vfio state:

```
struct KvmVfio {
  file_list: ListHead<KvmVfioFile>,
  lock: Mutex<()>,
  noncoherent: bool,                              // current coherency state
}

struct KvmVfioFile {
  link: ListLink,
  file: &Mutex<File>,                             // VFIO file
  iommu_group: Option<&IommuGroup>,                // ppc-only
}
```

Per-VM:

```
struct KvmArch {
  ...
  noncoherent_dma_count: AtomicU32,
}
```

`Vfio::file_is_valid(file) -> bool`:
1. Return file.f_op == VFIO_FILE_OPS.

`Vfio::file_set_kvm(file, kvm)`:
1. Per-vfio-API: vfio_file_set_kvm(file, kvm).

`Vfio::file_enforced_coherent(file) -> bool`:
1. Return vfio_file_enforced_coherent_dma(file).

`Vfio::file_add(dev, fd) -> Result<()>`:
1. filp = fget(fd).
2. If !file_is_valid(filp): fput; return Err(EINVAL).
3. Check filp not already in dev.file_list: return Err(EEXIST).
4. Allocate kvf = KvmVfioFile { file: filp, iommu_group: None }.
5. file_set_kvm(filp, dev.kvm).
6. mutex_lock(&dev.lock).
7. List_add(&kvf.link, &dev.file_list).
8. mutex_unlock.
9. Vfio::update_coherency(dev).

`Vfio::file_del(dev, fd) -> Result<()>`:
1. mutex_lock(&dev.lock).
2. Iterate dev.file_list: find kvf with kvf.file.fd == fd.
3. If found: list_del; file_set_kvm(kvf.file, NULL); fput; kfree(kvf).
4. mutex_unlock.
5. Vfio::update_coherency(dev).

`Vfio::update_coherency(dev)`:
1. mutex_lock(&dev.lock).
2. noncoherent_now = false.
3. For kvf in dev.file_list:
   - If !file_enforced_coherent(kvf.file): noncoherent_now = true; break.
4. If noncoherent_now != dev.noncoherent:
   - If noncoherent_now: kvm_arch_register_noncoherent_dma(dev.kvm).
   - Else: kvm_arch_unregister_noncoherent_dma(dev.kvm).
   - dev.noncoherent = noncoherent_now.
5. mutex_unlock.

`Vfio::set_file(dev, attr, arg) -> Result<()>`:
1. switch attr:
   - KVM_DEV_VFIO_FILE_ADD: copy_from_user(fd); return file_add(dev, fd).
   - KVM_DEV_VFIO_FILE_DEL: copy_from_user(fd); return file_del(dev, fd).
   - KVM_DEV_VFIO_FILE_SET_SPAPR_TCE: ppc-only; v0 unsupported.

`KvmArch::register_noncoherent_dma(kvm)`:
1. Per-x86: kvm.arch.noncoherent_dma_count++.
2. Per-vCPU mark for EPT memtype recompute.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `file_validity_checked_pre_add` | INVARIANT | per-add: file_is_valid first. |
| `unique_file_per_vm` | INVARIANT | per-vm.file_list at most one entry per file. |
| `noncoherent_count_balanced` | INVARIANT | per-add+del paired ⟹ count returns to 0. |
| `vfio_lock_held_during_list_mut` | INVARIANT | per-list-mut: dev.lock held. |
| `arch_register_iff_noncoherent_now` | INVARIANT | post-update_coherency: register_noncoherent iff dev.noncoherent_now. |

### Layer 2: TLA+

`virt/kvm/vfio.tla`:
- Per-VM file-add/del + coherency recompute.
- Properties:
  - `safety_no_double_register` — per-VM register_noncoherent paired with unregister.
  - `safety_pat_ignore_off_when_noncoherent` — noncoherent_dma_count > 0 ⟹ PAT-quirk-disabled.
  - `liveness_destroy_clears_files` — per-VM destroy ⟹ all kvm_vfio_files removed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vfio::file_add` post: kvf in list; file bound to kvm | `Vfio::file_add` |
| `Vfio::file_del` post: kvf removed; file unbound | `Vfio::file_del` |
| `Vfio::update_coherency` post: arch.noncoherent_dma_count reflects file_list state | `Vfio::update_coherency` |
| `Vfio::set_file` post: per-attr dispatched correctly | `Vfio::set_file` |

### Layer 4: Verus/Creusot functional

`Per-VFIO container with non-coherent DMA → kvm.arch.noncoherent_dma_count > 0 → vmx_ignore_guest_pat = false → EPT memtype = guest-PAT-resolved → assigned-device DMA sees correct memtype` semantic equivalence: per-VM PAT-quirk gating matches DMA-coherency requirement.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

VFIO-specific reinforcement:

- **Per-VFIO-fd validity check (file_is_valid)** — defense against per-arbitrary-fd binding non-VFIO file.
- **Per-VM file-list under dev.lock** — defense against concurrent add/del corruption.
- **Per-bind file_set_kvm propagates to VFIO** — defense against per-stale-fd seeing wrong kvm.
- **Per-coherency recompute on add+del** — defense against PAT-ignore-quirk lagging.
- **Per-arch_register_noncoherent_dma idempotent** — defense against per-bump under race.
- **Per-VM destroy iterates file-list** — defense against UAF via lingering bound file.
- **Per-add returns -EEXIST on duplicate** — defense against per-add double-bump.
- **Per-VFIO-file fget()/fput refcount balanced** — defense against per-file leak/UAF.
- **Per-SPAPR_TCE ppc-only at v0** — defense against per-arch unsupported attribute.
- **Per-attr KVM_DEV_VFIO_FILE_ADD/DEL/SET_SPAPR_TCE validated** — defense against per-attr injection.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — KVM_DEV_VFIO_* attribute payloads bounded; copy_from_user size-checked.
- **PAX_KERNEXEC** — kvm_device_ops table RO after registration.
- **PAX_RANDKSTACK** — randomized kstack on every KVM_CREATE_DEVICE / SET_DEVICE_ATTR.
- **PAX_REFCOUNT** — VFIO-file `fget` reference counted; saturating to prevent fput-after-free.
- **PAX_MEMORY_SANITIZE** — kvm_vfio_group / file-list nodes zeroed on detach.
- **PAX_UDEREF** — attr.addr validated as user pointer before copy_from_user.
- **PAX_RAP / kCFI** — device-op dispatch (set_attr/get_attr/destroy) type-checked.
- **GRKERNSEC_HIDESYM** — kvm/file pointers redacted in error paths.
- **GRKERNSEC_DMESG** — VFIO bind failures rate-limited.

VFIO-specific:

- **CAP_SYS_ADMIN strict on KVM_CREATE_DEVICE(VFIO)** — defense against unprivileged DMA binding.
- **file_set_kvm hook indirect-call type-checked** — defense against forged op-vector.
- **noncoherent_dma counter saturating** — overflow → panic instead of stale coherency.

Rationale: VFIO bind hands KVM control over DMA-capable device files; refcount/sanitize discipline blocks dangling-file UAF that would let a torn-down VM smuggle DMA into a fresh one.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VFIO core (drivers/vfio/; covered separately)
- Posted Interrupts (covered in `x86-posted-intr.md` Tier-3)
- IOMMU subsystem (covered separately)
- ppc SPAPR TCE (out-of-scope for x86_64 v0)
- Implementation code
