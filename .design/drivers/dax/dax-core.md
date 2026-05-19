# Tier-3: drivers/dax/{super,bus,device}.c — DAX devices, fsdax + devdax, direct-access page-frame, daxctl

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/dax/00-overview.md
upstream-paths:
  - drivers/dax/super.c
  - drivers/dax/bus.c
  - drivers/dax/bus.h
  - drivers/dax/device.c
  - drivers/dax/dax-private.h
  - drivers/dax/kmem.c
  - drivers/dax/pmem.c
  - drivers/dax/cxl.c
  - drivers/dax/hmem/hmem.c
  - drivers/dax/fsdev.c
  - include/linux/dax.h
-->

## Summary

The DAX (Direct Access) subsystem provides two complementary access modes onto byte-addressable persistent / volatile memory exposed by `pmem` / `cxl_pmem` / `cxl_kmem` / `hmem` (HMAT-discovered High-bandwidth memory) / `virtio_pmem` providers: **fsdax** — a filesystem mode where the filesystem (ext4 / XFS / virtio_fs) maps page-cache-free file pages directly onto pfn-backed PMEM/CXL ranges with `IS_DAX(inode)` semantics (no buffered-I/O bounce); and **devdax** — a per-region character device (`/dev/daxN.M`) that lets userspace `mmap` a contiguous, aligned PMEM/HBM extent directly without a filesystem, useful for SPDK/DPDK/userspace KV stores, persistent malloc, and KVM guest backends. A third path — **dax_kmem** — converts a devdax instance into kernel-managed `System RAM` via memory hotplug, useful for CXL.mem and HMAT-tiered HBM onlining.

The framework is layered as: **`dax_device` (super.c)** — the anchor inode + cdev + per-device `dax_operations` (`direct_access`, `zero_page_range`, `recovery_write`) + holder ops (`notify_failure`); **`dax_region` (bus.c)** — a per-provider SPA window with alignment + resource tree (managed by `dax_region_rwsem`); **`dev_dax` (device.c, dax-private.h)** — a per-region subdivision, the device-model object backing `/dev/daxN.M`, owns a `dev_pagemap` (driver-owned) that vmemmap-allocates `struct page` for the SPA range (either in DRAM or in-PMEM via `memmap=device`), exposes `mmap` / `fault` for devdax consumers. A `dax_bus` matches `dev_dax` against `dax_device_driver` instances by name or by `dax_kmem`-type override (`do_id_store`); `dax_match_type` decides between `DAXDRV_DEVICE_TYPE` (devdax) and `DAXDRV_KMEM_TYPE` (kmem-onlining).

This Tier-3 covers `drivers/dax/super.c` (~520 lines: anchor inode, ops table, SRCU read lock, fs_dax_get_by_bdev / fs_put_dax bridge), `drivers/dax/bus.c` (~1700 lines: dax_region + dev_dax bus matching + sysfs + size/align/range stores), `drivers/dax/device.c` (~600 lines: cdev + mmap + per-PTE/PMD/PUD fault handling). Cross-ref `mm/memremap.md` for `dev_pagemap` and `mm/page_alloc.md` for kmem onlining.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dax_device` | anchor object (inode, cdev, ops, holder_data) | `drivers::dax::DaxDevice` |
| `struct dax_operations` | `direct_access`, `zero_page_range`, `recovery_write` | `drivers::dax::Ops` |
| `struct dax_holder_operations` | `notify_failure` from PMEM provider to FS holder | `drivers::dax::HolderOps` |
| `struct dax_region` | per-provider SPA window (id, target_node, align, res, seed, youngest) | `drivers::dax::DaxRegion` |
| `struct dev_dax` | per-region subdivision (cdev backing `/dev/daxN.M`) | `drivers::dax::DevDax` |
| `struct dev_dax_range` | tuple `(pgoff, range, mapping)` for multi-range devdax | `drivers::dax::DevDaxRange` |
| `struct dax_mapping` | sysfs-visible per-range mapping object | `drivers::dax::DaxMapping` |
| `alloc_dax(private, ops)` / `put_dax(dax_dev)` / `kill_dax(dax_dev)` | anchor lifecycle | `DaxDevice::alloc` / `_put` / `_kill` |
| `dax_add_host(dax_dev, disk)` / `dax_remove_host(disk)` | bdev → dax_device map (xarray) | `DaxDevice::add_host` / `_remove_host` |
| `fs_dax_get_by_bdev(bdev, &off, holder, ops)` / `fs_put_dax(dax_dev, holder)` | filesystem fsdax handshake | `DaxDevice::fs_get_by_bdev` |
| `dax_read_lock()` / `dax_read_unlock(id)` (SRCU) | reader protection across lifecycle | `DaxDevice::read_*` |
| `dax_region_create(parent_dev, id, target_node, align, res)` | region constructor | `DaxRegion::create` |
| `__devm_create_dev_dax(...)` | dev_dax constructor (devm-managed) | `DevDax::create` |
| `dax_pgoff_to_phys(dev_dax, pgoff, size)` | translate file-offset → SPA | `DevDax::pgoff_to_phys` |
| `dev_dax_probe(dev_dax)` / `dev_dax_remove(dev_dax)` | devdax driver bind/unbind | `DevDax::probe` / `_remove` |
| `dax_open(inode, file)` / `dax_release(inode, file)` / `dax_mmap(file, vma)` | cdev fops | `DevDax::open` / `_release` / `_mmap` |
| `__dev_dax_pte_fault(dev_dax, vmf)` / `_pmd_fault(...)` / `_pud_fault(...)` | fault path per page size | `DevDax::fault_*` |
| `dev_dax_kmem_probe(dev_dax)` / `_remove(dev_dax)` | dax_kmem online-as-RAM driver | `DaxKmem::probe` / `_remove` |
| `dax_iomap_rw` / `dax_iomap_fault` (in `fs/dax.c`) | fsdax iomap consumer (cross-ref `fs/iomap.md`) | n/a — fs/dax owner |
| `dax_holder` / `dax_alive(dax_dev)` / `dax_synchronous(dax_dev)` | anchor state queries | `DaxDevice::holder` / `_alive` / `_synchronous` |

## Compatibility contract

REQ-1: `dax_device` is an anchor on a pseudo filesystem (`dax_mnt`) backed by `dax_superblock`; per-instance `inode` + `cdev` accessed via `inode_dax` / `dax_inode` helpers; lifetime managed by `dax_srcu` SRCU + per-inode refcount.

REQ-2: Per-`dax_region` alignment ∈ `{PAGE_SIZE, PMD_SIZE, PUD_SIZE}` per `dax_align_valid`; PUD requires `CONFIG_HAVE_ARCH_TRANSPARENT_HUGEPAGE_PUD`; PMD requires `has_transparent_hugepage()`.

REQ-3: `dax_region->res` is the parent resource tree under `dax_regions` (`DEFINE_RES_MEM_NAMED(0, -1, "DAX Regions")`); per-`dev_dax` sub-resources allocated under it via `dax_region_rwsem` write-held updates.

REQ-4: dev_dax can hold multiple ranges (`dev_dax->ranges[]`) representing a discontiguous SPA collection that the device backs as a contiguous file offset space via `pgoff`-keyed lookup in `dax_pgoff_to_phys`.

REQ-5: dev_dax bus matching: `dax_match_id` (per-name list registered via `new_id` / `remove_id` sysfs) OR `dax_match_type` (device-type vs kmem-type based on `IORESOURCE_DAX_KMEM` flag); `DAXDRV_DEVICE_TYPE` is default if `CONFIG_DEV_DAX_KMEM=n`.

REQ-6: Cdev fops permit only shared mappings (`VMA_MAYSHARE_BIT` required) — private (CoW) mmaps refused with `-EINVAL` to prevent dirty-page accounting confusion on direct-mapped storage.

REQ-7: Cdev mmap requires VMA start + end aligned to `dev_dax->align - 1` mask; misaligned mappings refused; faults at unsupported page sizes return `VM_FAULT_SIGBUS`.

REQ-8: PTE/PMD/PUD fault handlers each: `check_vma` (alignment + DAX file check), `dax_pgoff_to_phys(dev_dax, vmf->pgoff, fault_size)` to translate, allocate pfn via `pgmap`, install into pgtable via `vmf_insert_*`.

REQ-9: `dax_holder_operations.notify_failure(dax_dev, offset, len, mf_flags)` propagates per-SPA poison-discovery from a PMEM provider to the holder filesystem so that file-level metadata can be invalidated.

REQ-10: `dax_kmem` driver online-as-RAM path: probe iterates `dev_dax->ranges[]`, calls `add_memory_driver_managed(target_node, range.start, range_len(&range), "kmem", MHP_*)`; remove path inverses with `remove_memory`.

REQ-11: SRCU-based reader protection: `dax_read_lock()` returns a cookie used to bound `dax_alive`-checked sections; `kill_dax` synchronizes SRCU to drain readers before freeing.

REQ-12: Two global rwsems: `dax_region_rwsem` (per-region config changes), `dax_dev_rwsem` (per-device config changes including size/align/mapping list).

## Acceptance Criteria

- [ ] AC-1: `daxctl list -RD` enumerates `daxN` regions with target_node + align + size + per-device child list.
- [ ] AC-2: `daxctl reconfigure-device daxN.M --mode=devdax` produces `/dev/daxN.M`; `mmap` of an aligned region succeeds; private mmap (`MAP_PRIVATE`) refused with `-EINVAL`.
- [ ] AC-3: `daxctl reconfigure-device daxN.M --mode=system-ram` triggers `dax_kmem` bind + `add_memory_driver_managed` + new NUMA-tier ONLINE per memory-tier policy.
- [ ] AC-4: fsdax-mounted ext4 (`-o dax=always`) shows `IS_DAX(inode) == true`; `mmap(MAP_SHARED)` of an fsdax file produces direct-mapped pages observable via `/proc/PID/pagemap`.
- [ ] AC-5: `notify_failure` injected via debugfs lifts a per-SPA poison report into the holder filesystem; affected file produces `SIGBUS` on subsequent access.
- [ ] AC-6: Misaligned `mmap` against `daxN.M` with `align = PMD_SIZE` and `start = PAGE_SIZE` returns `-EINVAL`.
- [ ] AC-7: kselftest `tools/testing/selftests/dax/` exercises devdax open / mmap / fault / unmap on `dax_test` mocked region.

## Architecture

`DaxDevice` anchor (`drivers::dax::DaxDevice`):

```
struct DaxDevice {
  inode: Inode,                       // on dax_mnt pseudo-FS
  cdev: Cdev,                         // /dev/daxN.M
  private: *mut c_void,               // driver private (dev_dax)
  flags: DaxFlags,                    // ALIVE / SYNCHRONOUS / WRITE_CACHE / NOCACHE / NOMC
  ops: &'static DaxOps,
  holder_data: *mut c_void,           // FS holder (e.g. struct super_block)
  holder_ops: Option<&'static HolderOps>,
}
```

`DaxRegion` per-provider (`drivers::dax::DaxRegion`):

```
struct DaxRegion {
  id: i32,
  target_node: i32,                   // NUMA node when onlined
  kref: Refcount,
  dev: &'static Device,               // parent (pmem, cxl, hmem)
  align: u32,                         // PAGE_SIZE | PMD_SIZE | PUD_SIZE
  ida: Ida,                           // per-region instance allocator
  res: Resource,                      // SPA window
  seed: Option<&'static Device>,      // first unbound seed
  youngest: Option<&'static Device>,  // most recently created
}
```

`DevDax` per-region instance (`drivers::dax::DevDax`):

```
struct DevDax {
  region: Arc<DaxRegion>,
  dax_dev: Arc<DaxDevice>,
  virt_addr: *mut c_void,             // kva from memremap (fsdev only)
  cached_size: u64,
  align: u32,
  target_node: i32,
  dyn_id: bool,
  id: i32,
  ida: Ida,                           // per-instance mapping ID allocator
  dev: Device,
  pgmap: *mut DevPagemap,             // driver-owned; vmemmap-allocates struct page
  memmap_on_memory: bool,             // place memmap in-PMEM instead of DRAM
  nr_range: u32,
  ranges: KVec<DevDaxRange>,          // (pgoff, range, mapping)
}
```

Region create (`dax_region_create`, called by `pmem.c` / `cxl.c` / `hmem.c` providers):
1. Allocate `dax_region` under `dax_regions` parent resource.
2. Insert SPA window as a sub-resource (`request_resource`).
3. Initialize per-region IDA + seed pointers.
4. Register sysfs nodes (`region_size`, `align`, `available_size`, `create`, `delete`, `seed`, `id`).

dev_dax create (`__devm_create_dev_dax`):
1. Acquire `dax_region_rwsem` write.
2. Allocate `dev_dax` + cdev.
3. Allocate id via region IDA (or static id from caller).
4. Allocate sub-resource(s) from region's res tree (one per range).
5. Initialize `dev_pagemap` (driver-owned, type `MEMORY_DEVICE_GENERIC` for devdax / `_PRIVATE` for kmem).
6. `alloc_dax(dev_dax, &dev_dax_ops)` builds anchor; `cdev_add` registers cdev.
7. Register sysfs nodes (`size`, `align`, `target_node`, `mapping`, `numa_node`).

Bus matching (`dax_bus_match`):
1. `dax_match_id` — name match against per-driver `dax_id` list (populated via `new_id` / `remove_id`).
2. `dax_match_type` — type match (`DAXDRV_DEVICE_TYPE` vs `_KMEM_TYPE`); kmem flag derived from `dev_dax->region->res.flags & IORESOURCE_DAX_KMEM`.
3. Driver `probe` invoked: `dev_dax_probe` (devdax cdev) or `dev_dax_kmem_probe` (memory hotplug).

Cdev fault path (`__dev_dax_pte_fault` / `_pmd_fault` / `_pud_fault`):
1. `check_vma(dev_dax, vmf->vma, __func__)` — verifies `dax_alive`, shared-only, alignment, DAX-file.
2. If `dev_dax->align > fault_size` (e.g. PMD-aligned region, PTE fault): return `VM_FAULT_SIGBUS`.
3. `dax_pgoff_to_phys(dev_dax, vmf->pgoff, fault_size)` translates file offset → SPA via the `ranges[]` lookup.
4. Convert phys → pfn; `pgmap`-backed struct page found via `pfn_to_page`.
5. `dax_set_mapping(vmf, pfn, fault_size)` populates `folio->mapping` + `folio->index` for `nr_pages` pages.
6. `vmf_insert_pfn` / `vmf_insert_pfn_pmd` / `vmf_insert_pfn_pud` installs PTE/PMD/PUD entry.

dax_kmem onlining (`dev_dax_kmem_probe`):
1. Iterate `dev_dax->ranges[]`.
2. For each: `add_memory_driver_managed(target_node, range.start, range_len, "kmem", mhp_flags)`.
3. Per-tier policy (cross-ref `mm/memory-tiers.md`) demotes node into appropriate tier based on per-region access_coordinates.
4. On error: roll back via `remove_memory` for already-added ranges; release sub-resource.

SRCU lifecycle:
- Every reader: `id = dax_read_lock()` → uses `dax_alive(dax_dev)` to check + uses ops → `dax_read_unlock(id)`.
- `kill_dax(dax_dev)` clears `DAXDEV_ALIVE` then `synchronize_srcu(&dax_srcu)` to drain readers before `put_dax`.

## Hardening

- **Cdev mmap shared-only** — `__check_vma` refuses private mappings to prevent dirty-page accounting confusion against direct-mapped storage.
- **Alignment-enforced VMA + fault** — VMA start/end + fault address must match `dev_dax->align`; misaligned → `-EINVAL` (mmap) or `VM_FAULT_SIGBUS` (fault).
- **`dax_alive` checked under SRCU** — defense against use-after-`kill_dax`.
- **Per-region rwsem** — coordinates concurrent reconfigure (size / align / range list) against ongoing mmap/fault.
- **`fs_dax_get_by_bdev` partition alignment** — refuses to expose a partition whose start_off or length is not PAGE-aligned.
- **Pageframe via driver-owned `pgmap`** — `dev_pagemap` cleanup via percpu_ref drain blocks until last reference drops.
- **`vmf_insert_pfn_pmd` / `_pud` require correct CPU + region alignment** — defense against misaligned huge mapping crash.
- **`notify_failure` holder check** — only the registered holder receives failure notifications.
- **`dax_match_type` kmem-only opt-in** — `CONFIG_DEV_DAX_KMEM` must be enabled; without it, devdax mode is forced.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `dax_device`, `dax_region`, `dev_dax`, `dev_dax_range`, `dax_mapping`, `dax_id`; refuse userspace copies outside cdev fops + sysfs helpers.
- **PAX_KERNEXEC** — DAX core (`alloc_dax`, `kill_dax`, `dax_region_create`, `__devm_create_dev_dax`, `dev_dax_probe`, cdev fops, fault handlers) in `__ro_after_init` text with W^X enforced.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across cdev open / mmap / fault / unmap and dax_kmem probe entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on `dax_device` (inode refcount), `dax_region` (kref), `dev_dax`, `dev_pagemap` (percpu_ref); overflow trap defeats reconfigure-vs-mmap race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `dax_region` resource records, `dev_dax` range arrays, per-mapping cached sizes so stale SPA ranges cannot bleed into a reused region slot; cross-ref RAM-mode poison cleared before next devdax reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every cdev IOCTL / sysfs entry; refuse user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `dax_operations` (`direct_access`, `zero_page_range`, `recovery_write`), `dax_holder_operations.notify_failure`, cdev fops, fault-handler vtable marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-region / per-dev_dax pointer disclosure behind CAP_SYSLOG; suppress `%p` in dax tracepoints.
- **GRKERNSEC_DMESG** — restrict misaligned-mmap, alive-check-fail, notify_failure, kmem-onlining-fail banners to CAP_SYSLOG.
- **`/dev/dax*` CAP_SYS_RAWIO** — cdev open requires CAP_SYS_RAWIO in the owning user namespace; `dev_dax` cdev nodes default `mode = 0600 root:root`; udev rules must opt-in to grant per-uid access.
- **mmap of dax-device PAX_KERNEXEC (NX-bit force)** — devdax-backed VMA installations force `VM_NOEXEC` regardless of `prot` argument; refuse `PROT_EXEC` outright in `dax_mmap` (PMEM/HBM is not executable kernel-managed text). Cross-ref hardware NX-bit forced on the mapped PTEs/PMDs/PUDs.
- **devdax open lockdown** — when `kernel_lockdown` is in `confidentiality` mode, cdev open refused (PMEM contents are typically privileged); when in `integrity` mode, `MAP_SHARED + PROT_WRITE` refused on persistent-mode devdax.
- **Reconfigure CAP_SYS_ADMIN** — every `_store` writer (size / align / mapping / mode toggles) requires CAP_SYS_ADMIN in init userns; refuse from non-init userns.
- **dax_kmem opt-in audited** — every `dev_dax_kmem_probe` success emits a structured audit record (region, range, target_node) so unexpected kmem-onlining is observable.
- **`notify_failure` holder-only** — refuse to deliver failure notifications to any non-registered holder; defense against poison-disclosure attacks.
- **SRCU drain on `kill_dax`** — refuse to free dax_dev until SRCU drained; defense against reader-vs-killer UAF.

Rationale: `/dev/dax*` is a direct, byte-addressable handle to potentially-persistent memory that may contain at-rest credentials, KV-store data, or kernel-online RAM. Allowing unprivileged open is a confidentiality + integrity vulnerability; allowing executable mmap turns a memory-pool node into a W^X-bypass primitive; allowing private mappings produces accounting confusion. CAP_SYS_RAWIO on open, mandatory NX on mmap, lockdown integration, audited kmem-onlining, SRCU-bounded readers, and refcount saturation on `dev_pagemap` make DAX a mediated boundary rather than a memory-disclosure passthrough.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pgoff_to_phys_no_oob` | OOB | `dax_pgoff_to_phys(dev_dax, pgoff, sz)` returns valid SPA only when `pgoff + sz/PAGE_SIZE` ≤ sum of `range_len(&ranges[i])`. |
| `mmap_alignment_total` | TOTALITY | `__check_vma` rejects every `start`/`end` not aligned to `dev_dax->align - 1` mask. |
| `fault_size_vs_align` | TOTALITY | PTE-fault refused when `dev_dax->align > PAGE_SIZE`; PMD-fault refused when `align > PMD_SIZE`; PUD-fault similarly. |
| `dax_alive_srcu` | UAF | every `dax_operations` dispatch wrapped by `dax_read_lock` + `dax_alive` check; `kill_dax` synchronizes SRCU. |

### Layer 2: TLA+

`models/dax/lifecycle.tla` (parent-declared): models concurrent `alloc_dax` / cdev-open / mmap / fault / `kill_dax` under SRCU; proves no fault executes against a killed dax_dev.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `__check_vma` post: on `Ok`, `vma->vm_start ≡ 0 (mod align) ∧ vma->vm_end ≡ 0 (mod align) ∧ file_is_dax(vma->vm_file)`. | `device` |
| `dax_set_mapping` post: every folio in `[head, head + nr_pages)` has `folio->mapping == filp->f_mapping ∧ folio->index ∈ {pgoff, pgoff+1, ...}`. | `device` |
| `__devm_create_dev_dax` post: per-range sub-resource is a subset of `dax_region->res` AND non-overlapping with sibling devices. | `bus` |
| `dev_dax_kmem_probe` post: on `Ok`, every `add_memory_driver_managed` succeeded AND target_node bound; on `Err`, all previously-added ranges have `remove_memory` invoked. | `kmem` |

### Layer 4: Verus/Creusot functional

End-to-end: userspace `open("/dev/daxN.M") → mmap(MAP_SHARED, len, PROT_READ|PROT_WRITE) → store/load` produces direct CPU access to the SPA range advertised by the provider's `dax_region`; encoded as a Verus refinement from cdev open through fault dispatch to PTE/PMD/PUD installation matching the `pgmap`'s SPA → pfn function.

## Out of Scope

- `fs/dax.c` iomap consumers (covered in `fs/iomap.md` + `fs/dax.md` future Tier-3)
- per-provider drivers (`drivers/dax/pmem.c`, `drivers/dax/cxl.c`, `drivers/dax/hmem/hmem.c`): Tier-4 if needed
- `dev_pagemap` + memremap_pages internals (covered in `mm/memremap.md`)
- Memory-tier policy + auto-demotion (covered in `mm/memory-tiers.md`)
- KVM guest-backend usage of devdax
