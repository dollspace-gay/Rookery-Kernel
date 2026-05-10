# Tier-3: virt/kvm/guest_memfd.c — KVM guest_memfd (private guest memory for confidential VMs)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - virt/kvm/guest_memfd.c (~1030 lines)
  - include/linux/kvm_host.h (kvm_gmem_*)
  - include/uapi/linux/kvm.h (KVM_CREATE_GUEST_MEMFD)
-->

## Summary

`guest_memfd` (gmem) is the kernel-managed memory backing for confidential VMs (TDX, SEV-SNP, future Arm CCA) where guest memory must NOT be mappable into userspace VMM (QEMU). Per-VM creates a gmem-fd via KVM_CREATE_GUEST_MEMFD ioctl returning anon-inode fd; the inode is bound to per-memslot via `gmem_offset` in `KVM_SET_USER_MEMORY_REGION2`. KVM access via `kvm_gmem_get_pfn` (kernel-only); userspace cannot mmap. Per-page tracked private vs shared via per-VM attribute bitmap (`KVM_MEMORY_ATTRIBUTE_PRIVATE`). Page-eviction triggers MMU-notifier-style invalidate for KVM's per-memslot rmap. Critical for: TDX/SEV-SNP confidential computing.

This Tier-3 covers `guest_memfd.c` (~1030 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct gmem_file` | per-fd state | `GmemFile` |
| `kvm_gmem_create()` | per-VM create gmem | `Gmem::create` |
| `__kvm_gmem_create()` | internal alloc | `Gmem::__create` |
| `kvm_gmem_bind()` | per-memslot bind | `Gmem::bind` |
| `kvm_gmem_unbind()` | per-memslot unbind | `Gmem::unbind` |
| `kvm_gmem_get_pfn()` | per-(slot, idx) pfn lookup | `Gmem::get_pfn` |
| `__kvm_gmem_get_pfn()` | internal pfn-lookup | `Gmem::__get_pfn` |
| `kvm_gmem_invalidate_begin/end()` | per-page invalidate notifier | `Gmem::invalidate_*` |
| `gmem_punch_hole()` | per-(start, end) eviction | `Gmem::punch_hole` |
| `KVM_CREATE_GUEST_MEMFD` | UAPI ioctl | UAPI |
| `KVM_GUEST_MEMFD_ALLOW_HUGEPAGE` | per-flag | UAPI |
| `KVM_MEMORY_ATTRIBUTE_PRIVATE` | per-page attr | UAPI |
| `kvm_create_guest_memfd` | UAPI struct | UAPI |
| `KVM_SET_MEMORY_ATTRIBUTES` | per-VM private/shared toggle | UAPI |
| `KVM_SET_USER_MEMORY_REGION2` | per-memslot register w/ gmem | UAPI |

## Compatibility contract

REQ-1: KVM_CREATE_GUEST_MEMFD ioctl:
- args: { size, flags, reserved[6] }.
- Returns fd to anon-inode (gmem-fd).
- Per-fd inode mapped to backing folios.

REQ-2: Per-fd inode:
- Anon-inode (no userspace path).
- inode_operations subset: only kernel-internal access.
- Per-folio allocated on first kvm_gmem_get_pfn.

REQ-3: KVM_SET_USER_MEMORY_REGION2 with gmem:
- struct kvm_userspace_memory_region2 has gmem_fd + gmem_offset.
- Per-memslot binds gmem-fd at gmem_offset for `memory_size`.
- Per-vCPU access uses kvm_gmem_get_pfn instead of get_user_pages.

REQ-4: kvm_gmem_get_pfn(kvm, slot, gfn, pfn, max_order):
- Lookup file = slot.gmem_file.
- index = gfn - slot.base_gfn + slot.gmem_offset.
- folio = filemap-or-alloc(file, index).
- *pfn = folio_pfn(folio) + (gfn & (folio_nr_pages - 1)).
- *max_order = folio_order(folio).

REQ-5: Per-page invalidate:
- Per-folio truncate / punch_hole / migration:
  - kvm_gmem_invalidate_begin(inode, start, end): walks per-VM memslots; KVM-flushes EPT/NPT for affected GFNs.
  - kvm_gmem_invalidate_end(inode, start, end): post-invalidate cleanup.

REQ-6: Per-page private/shared:
- Per-VM kvm.mem_attr_array tracks per-GFN attributes.
- KVM_MEMORY_ATTRIBUTE_PRIVATE bit: requires gmem.
- Per-shared GFN: traditional userspace HVA.
- Per-toggle: VMM converts via KVM_SET_MEMORY_ATTRIBUTES.

REQ-7: Per-fd lifetime:
- Held by VMM (via fd refcount).
- Per-VM holds reference via memslot binding.
- Per-VM destroy: kvm_gmem_unbind on each bound memslot.

REQ-8: Per-folio order:
- Default: PAGE_SIZE (order 0).
- KVM_GUEST_MEMFD_ALLOW_HUGEPAGE flag: allow PMD-sized (2MB) folios for SEPT large-page mapping.

REQ-9: Per-fd vs userspace mmap:
- gmem-fd: userspace cannot mmap (file_operations.mmap = NULL).
- Kernel-only access via kvm_gmem_get_pfn.

REQ-10: Per-page poison:
- Per-folio poisoned (HWERR): kvm_gmem_get_pfn returns -EHWPOISON.
- VMM should retry or kill VM.

REQ-11: TDX integration:
- Per-page TDX-accept: TD module owns the EPT mapping.
- KVM updates SEPT entries based on guest PTE faults.

REQ-12: SEV-SNP integration:
- Per-page RMP-checked at fault.
- Per-vCPU PVALIDATE handled by guest.

## Acceptance Criteria

- [ ] AC-1: KVM_CREATE_GUEST_MEMFD(size=4GB): returns fd; size visible via fstat.
- [ ] AC-2: KVM_SET_USER_MEMORY_REGION2 with gmem-fd: memslot.gmem_file set.
- [ ] AC-3: kvm_gmem_get_pfn(kvm, slot, gfn): returns valid pfn; folio allocated lazily.
- [ ] AC-4: VMM userspace mmap(gmem-fd): fails -ENOSYS.
- [ ] AC-5: KVM_SET_MEMORY_ATTRIBUTES(PRIVATE) on GFN: attr-bit set.
- [ ] AC-6: Per-folio truncate via fallocate(PUNCH_HOLE): kvm_gmem_invalidate_begin/end called.
- [ ] AC-7: Per-VM destroy: gmem-fd held by VMM; bound memslots unbound.
- [ ] AC-8: KVM_GUEST_MEMFD_ALLOW_HUGEPAGE: 2MB folios allocated for aligned ranges.
- [ ] AC-9: HWPOISON folio: kvm_gmem_get_pfn returns -EHWPOISON.
- [ ] AC-10: TDX boot: SEPT populated via gmem-fd faults.

## Architecture

Per-fd state:

```
struct GmemFile {
  inode: &Inode,                                 // anon-inode
  flags: u64,                                    // KVM_GUEST_MEMFD_*
  bindings: ListHead<GmemBinding>,               // per-VM memslot bindings
  bindings_lock: Mutex<()>,
}

struct GmemBinding {
  link: ListLink,
  kvm: &Kvm,
  slot: &KvmMemorySlot,
  start: PgoffT,                                 // gmem_offset >> PAGE_SHIFT
  end: PgoffT,
}
```

Per-VM memslot (extended for gmem):

```
struct KvmMemorySlot {
  ...
  gmem_file: Option<&GmemFile>,                   // None for traditional HVA-backed
  gmem_offset: u64,
  // Existing: base_gfn, npages, userspace_addr, dirty_bitmap, ...
}
```

Per-VM attr array:

```
struct KvmArch {
  ...
  mem_attr_array: XArray<u64>,                    // per-GFN attribute bitmap
}
```

`Gmem::create(kvm, args) -> Result<i32>`:
1. Validate args.size aligned to PAGE_SIZE.
2. Create anon-inode with gmem ops.
3. Allocate GmemFile.
4. Set inode.size = args.size.
5. Set f.flags = args.flags.
6. Return fd.

`Gmem::bind(kvm, slot, fd, gmem_offset) -> Result<()>`:
1. file = fget(fd).
2. Verify file.f_op == GMEM_OPS.
3. Allocate GmemBinding.
4. binding.kvm = kvm; binding.slot = slot.
5. binding.start = gmem_offset >> PAGE_SHIFT.
6. binding.end = binding.start + slot.npages.
7. mutex_lock(&file.f.bindings_lock).
8. list_add(&binding.link, &file.f.bindings).
9. mutex_unlock.
10. slot.gmem_file = file.f.
11. slot.gmem_offset = gmem_offset.

`Gmem::get_pfn(kvm, slot, gfn, pfn, max_order) -> Result<()>`:
1. file = slot.gmem_file.
2. index = (gfn - slot.base_gfn) + slot.gmem_offset >> PAGE_SHIFT.
3. folio = filemap_lookup_or_alloc(file, index).
4. If folio.poisoned: return Err(EHWPOISON).
5. *pfn = folio_pfn(folio) + (gfn & ((1 << folio.order) - 1)).
6. *max_order = folio.order.
7. Ok.

`Gmem::invalidate_begin(inode, start, end, attr_filter)`:
1. mutex_lock(&inode.f.bindings_lock).
2. For binding in inode.f.bindings:
   - If start < binding.end ∧ end > binding.start:
     - kvm = binding.kvm; slot = binding.slot.
     - For gfn in (max(start, binding.start) - binding.start + slot.base_gfn)..min(end, binding.end) - binding.start + slot.base_gfn:
       - kvm_arch_invalidate_memslot_gfn(kvm, slot, gfn) (flush EPT/NPT).
3. mutex_unlock.

`Gmem::invalidate_end(inode, start, end)`:
1. Symmetric end-of-invalidate cleanup.

`Gmem::punch_hole(inode, start, end)`:
1. Gmem::invalidate_begin(inode, start, end).
2. truncate_inode_pages_range(inode.mapping, start, end).
3. Gmem::invalidate_end(inode, start, end).

`Gmem::release(file)`:
1. mutex_lock(&file.f.bindings_lock).
2. For binding in file.f.bindings:
   - kvm_gmem_invalidate_begin/end (full range).
   - Remove from kvm.memslot.gmem_file.
3. mutex_unlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `gmem_size_aligned` | INVARIANT | per-create size aligned to PAGE_SIZE. |
| `gmem_no_userspace_mmap` | INVARIANT | gmem.f_op.mmap == NULL. |
| `binding_range_within_slot` | INVARIANT | per-binding: end - start == slot.npages. |
| `pfn_within_folio` | INVARIANT | per-get_pfn: pfn ∈ [folio_pfn, folio_pfn + folio_nr_pages). |
| `invalidate_under_bindings_lock` | INVARIANT | per-invalidate: bindings_lock held. |

### Layer 2: TLA+

`virt/kvm/gmem.tla`:
- Per-fd create + per-binding bind + per-get_pfn + per-invalidate.
- Properties:
  - `safety_no_userspace_access` — gmem-fd ⟹ userspace mmap fails.
  - `safety_invalidate_propagates_to_kvm` — per-folio-evict ⟹ KVM flushes EPT/NPT.
  - `liveness_get_pfn_succeeds` — per-non-poisoned page ⟹ kvm_gmem_get_pfn returns valid pfn.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Gmem::create` post: anon-inode created; size aligned | `Gmem::create` |
| `Gmem::bind` post: binding in file.bindings; slot.gmem_file set | `Gmem::bind` |
| `Gmem::get_pfn` post: pfn within folio range; max_order matches folio | `Gmem::get_pfn` |
| `Gmem::invalidate_begin` post: per-affected-binding KVM-flush invoked | `Gmem::invalidate_begin` |

### Layer 4: Verus/Creusot functional

`Per-confidential-VM gmem-fd → per-page kvm_gmem_get_pfn → folio lazy-alloc → kernel-only kva → VMM mmap fails → SEPT/NPT populated; per-folio invalidate → EPT/NPT flush` semantic equivalence: per-gmem matches confidential-computing memory-isolation requirement.

## Hardening

(Inherits row-1 features from `virt/kvm/memslots.md` § Hardening.)

guest_memfd-specific reinforcement:

- **Per-fd no userspace mmap** — defense against per-VMM-leak of confidential guest memory.
- **Per-folio order honored at SEPT** — defense against per-mapping-fragmentation reducing TDX huge-page benefit.
- **Per-bind requires gmem.f_op match** — defense against per-fd type confusion.
- **Per-invalidate flushes per-binding KVM EPT/NPT** — defense against per-folio-evict stale guest mapping.
- **Per-poisoned folio returns -EHWPOISON** — defense against per-corruption silent propagation.
- **Per-attribute KVM_MEMORY_ATTRIBUTE_PRIVATE persisted in xarray** — defense against per-conversion mishandle.
- **Per-VM destroy releases bindings** — defense against post-VM bindings leaking gmem refs.
- **Per-fd refcount via fget/fput** — defense against UAF.
- **Per-folio alloc lazily** — defense against per-init OOM on multi-GB gmem.
- **Per-shared/private boundary enforced** — defense against per-mismatch confidential leak.
- **Per-TDX SEPT update aligned with gmem invalidate** — defense against per-stale SEPT entry.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- KVM memslots (covered in `memslots.md` Tier-3)
- TDX core (covered in `x86-tdx.md` Tier-3)
- SEV core (covered in `x86-sev.md` Tier-3)
- Implementation code
