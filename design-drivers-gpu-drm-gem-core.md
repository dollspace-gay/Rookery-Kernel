---
title: "Tier-3: drivers/gpu/drm/drm_gem.c — GEM (Graphics Execution Manager) buffer-object framework"
tags: ["tier-3", "drivers-drm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

GEM (Graphics Execution Manager) is the per-DRM-driver buffer-object framework — every framebuffer, every texture, every command-buffer, every shared dma-buf imported from another driver, every userspace map of GPU memory passes through here. Per-driver `drm_gem_object` is the base class extended by per-backend specializations (TTM-managed VRAM, shmem-backed system memory, DMA-coherent contiguous, VRAM-helper for simple GPUs). Owns: per-object refcount lifecycle, per-FD handle table mapping (handle → object), per-object sysfs `mmap` offset allocator, dma-buf export + import via PRIME, vma offset manager.

This Tier-3 covers `drivers/gpu/drm/drm_gem.c` (~1800 lines: core), and references the per-helper files (drm_gem_dma_helper.c, drm_gem_shmem_helper.c, drm_gem_ttm_helper.c, drm_gem_vram_helper.c, drm_gem_atomic_helper.c, drm_gem_framebuffer_helper.c) which extend the core for specific backends.

### Acceptance Criteria

- [ ] AC-1: kmscube + Mesa work on amdgpu / i915 / xe / nouveau (PRIME-share GEM between Mesa+EGL and DRM modeset).
- [ ] AC-2: GBM (`gbm_bo_create`) allocates GEM object via per-driver dumb-create; reflected in `/sys/class/drm/...`.
- [ ] AC-3: Cross-driver dma-buf test: V4L2 capture into GEM-backed dma-buf, displayed via DRM atomic commit; zero-copy.
- [ ] AC-4: Per-fd handle stress: alloc 10000 GEM handles, close fd → all auto-freed; KASAN clean.
- [ ] AC-5: ww-mutex test: simultaneous atomic commits referencing overlapping GEM-set ww-mutex-converge correctly via wound-wait.
- [ ] AC-6: Dumb-buffer test: `DRM_IOCTL_MODE_CREATE_DUMB` + `_MAP_DUMB` + mmap → ARGB pixels writable.
- [ ] AC-7: PRIME export+re-import test: GPU-A exports GEM, GPU-B imports as foreign-fd; subsequent atomic commit on GPU-B references the imported buffer.

### Architecture

`Object` lives in `drm::gem::Object`:

```
struct Object {
  refcount: Refcount,
  dev: Arc<DrmDevice>,
  size: u64,
  filp: Option<Arc<File>>,            // shmem backing-file (None for non-shmem)
  vma_node: VmaOffsetNode,
  resv: KBox<DmaResv>,                 // reservation object (shared/exclusive fence list)
  funcs: Arc<dyn ObjectFuncs>,
  import_attach: Option<Arc<DmaBufAttachment>>,
  dma_buf: Option<Arc<DmaBuf>>,        // when exported (cached for re-export)
  handle_count: AtomicU32,             // per-fd handles count (sysfs visibility)
  rss: u64,                            // resident-set-size for cgroup accounting
  name: u32,                            // legacy GEM "flink" name (deprecated)
}
```

`File` (per-fd state, lives in `drm::file::File`):

```
struct File {
  refcount: Refcount,
  dev: Arc<DrmDevice>,
  pid: Pid,
  comm: KString,
  client_id: u64,
  authenticated: AtomicBool,
  is_master: AtomicBool,
  master: Option<Arc<DrmMaster>>,
  object_handles: Mutex<XArray<Arc<Object>>>,  // handle → object
  object_idr_mutex: Mutex<()>,
  prime: Mutex<DrmPrimeFile>,
  fbs: Mutex<Vec<Arc<DrmFramebuffer>>>,
  syncobj_handles: Mutex<XArray<Arc<DrmSyncObj>>>,
  ...
}
```

GEM object init `Object::init(dev, size)`:
1. `kref_init(&obj->refcount)`.
2. `dev->driver->gem_create_object(dev, size)` → per-driver creates per-driver-extended object.
3. `dev->vma_offset_manager` not yet allocated until `create_mmap_offset` called.
4. Embed `dma_resv` for sync.

Per-FD handle alloc `File::create_gem_handle(file, obj, &handle)`:
1. Take `file.object_idr_mutex`.
2. `xa_alloc(&file.object_handles, &handle, obj, 1..U32::MAX)` → assign 32-bit handle.
3. `obj.handle_count.fetch_add(1)`.
4. `Arc::clone(&obj)` (incremented refcount).

Per-FD handle lookup `File::lookup_gem_handle(file, handle)`:
1. `xa_load(&file.object_handles, handle)` → `Option<Arc<Object>>` (cloned).

PRIME export `File::handle_to_dmabuf_fd(file, handle, flags)`:
1. Look up obj.
2. If `obj.dma_buf.is_some()`: re-use cached dma_buf (just bump dma_buf refcount + create new fd).
3. Else: `obj.funcs.export(obj, flags)` (per-driver) → returns `Arc<DmaBuf>`; cache in `obj.dma_buf`.
4. `dma_buf_fd(dma_buf, flags)` → returns int fd to userspace.

PRIME import `File::dmabuf_fd_to_handle(file, prime_fd)`:
1. `dma_buf = dma_buf_get(prime_fd)`.
2. Walk `file.prime.dmabufs` for existing local-handle for this dma_buf; if found, return cached.
3. Else: `dev.driver.gem_prime_import(dev, dma_buf)` (per-driver) — typically:
   - `dma_buf_attach(dma_buf, &dev->dev)`.
   - Allocate per-driver GEM object wrapping the dmabuf.
   - `obj.import_attach = Some(attach)`.
4. `File::create_gem_handle(file, &obj, &handle)`.
5. Return handle.

Per-object mmap path `File::gem_mmap(file, vma)`:
1. Look up `Object` by `vma.vm_pgoff` via `dev.vma_offset_manager`.
2. Validate `vma.vm_end - vma.vm_start <= obj.size`.
3. `obj.funcs.mmap(obj, vma)` (per-driver) → typically:
   - For shmem-helper: install `vm_ops` that on page-fault calls `obj.funcs.get_pages(obj)` + `vmf_insert_mixed(vma, vmf->address, pfn)`.
   - For TTM-helper: TTM ttm_bo_vm_ops handles fault via `ttm_bo_vm_fault`.
   - For DMA-helper: `dma_mmap_attrs(...)` directly maps DMA-coherent region.

Per-object dma_resv (reservation) integration: per-object `obj.resv` holds list of pending exclusive + shared `dma_fence`s. GPU command-submit adds a fence to `obj.resv` representing in-flight GPU access. CPU read/write must `dma_resv_wait_unlocked(obj, exclusive=false, ...)` for shared fences (sync to GPU completion). DRM atomic commit (cross-ref `drm-atomic.md`) takes ww-mutex over per-CRTC framebuffer GEM objects' resvs.

ww-mutex chain `Object::lock_reservations(objs[N], &ctx)`:
1. Init AcquireCtx via `ww_acquire_init(&ctx, &reservation_ww_class)`.
2. For each obj in objs:
   - `ww_mutex_lock(obj.resv.lock, &ctx)`.
   - On -EDEADLK: rollback (unlock prior), backoff via `ww_mutex_lock_slow(...)`, restart.

Cross-driver dma-buf import: GPU-side import of V4L2-exported dma-buf becomes a GEM object wrapping the dma-buf; per-driver import code attaches via `dma_buf_attach(...)` + `dma_buf_map_attachment(...)` to populate sg_table from V4L2's underlying memory.

### Out of Scope

- TTM (covered in `drm-ttm.md` future Tier-3)
- Per-helper backends (covered in `drm-gem-shmem.md`, `drm-gem-dma.md`, `drm-gem-vram.md`, `drm-gem-ttm.md` future Tier-3s)
- Per-driver implementation (amdgpu / i915 / xe / nouveau / etc.)
- DMA-fence + DMA-resv impl (covered in `kernel/dma/dmabuf-core.md` future Tier-3)
- DRM atomic commit (covered in `drm-atomic.md` Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct drm_gem_object` | base GEM object | `drm::gem::Object` |
| `struct drm_gem_object_funcs` | per-driver vtable (`free`, `pin`, `unpin`, `get_sg_table`, `vmap`, `vunmap`, `mmap`, `evict`, `rss`, `print_info`) | `drm::gem::ObjectFuncs` (trait) |
| `drm_gem_object_init(dev, obj, size)` | init common GEM fields | `Object::init` |
| `drm_gem_private_object_init(dev, obj, size)` | private init for non-shmem-backed | `Object::private_init` |
| `drm_gem_object_release(obj)` | release common fields | `Object::release` |
| `drm_gem_object_get(obj)` / `_put(obj)` | refcount get/put | `Arc<Object>` get/put |
| `drm_gem_handle_create(file, obj, &handle)` | per-fd handle alloc | `File::create_gem_handle` |
| `drm_gem_handle_delete(file, handle)` | per-fd handle free | `File::delete_gem_handle` |
| `drm_gem_object_lookup(file, handle)` | per-fd handle → object | `File::lookup_gem_handle` |
| `drm_gem_create_mmap_offset(obj)` / `_size` | per-object mmap-offset alloc | `Object::create_mmap_offset` / `_size` |
| `drm_gem_free_mmap_offset(obj)` | inverse | `Object::free_mmap_offset` |
| `drm_gem_mmap(filp, vma)` | top-level mmap entry | `File::gem_mmap` |
| `drm_gem_mmap_obj(obj, obj_size, vma)` | per-object mmap | `Object::mmap_obj` |
| `drm_gem_prime_handle_to_fd(dev, file_priv, handle, flags, &prime_fd)` | export GEM as dma-buf-fd (PRIME export) | `File::handle_to_dmabuf_fd` |
| `drm_gem_prime_fd_to_handle(dev, file_priv, prime_fd, &handle)` | import dma-buf-fd as GEM handle (PRIME import) | `File::dmabuf_fd_to_handle` |
| `drm_gem_dmabuf_export(dev, exp_info)` | helper that calls `dma_buf_export` | `Object::export_dmabuf` |
| `drm_gem_prime_export(obj, flags)` | per-driver overridable export | `Object::prime_export` |
| `drm_gem_prime_import(dev, dma_buf)` | per-driver overridable import | `Object::prime_import` |
| `drm_gem_prime_import_dev(dev, dma_buf, attach_dev)` | import to specific device | `Object::prime_import_dev` |
| `drm_gem_evict(obj)` / `_evictable(obj)` | per-object evict + check | `Object::evict` / `_is_evictable` |
| `drm_gem_lock_reservations(...)` / `_unlock_reservations(...)` | acquire ww-mutex chain over multiple GEM objects | `Object::lock_reservations` / `_unlock_reservations` |
| `drm_gem_get_pages(obj)` / `_put_pages(obj, pages, dirty, accessed)` | shmem-backed pages | (shmem-helper) `Object::get_pages` / `_put_pages` |
| `drm_gem_dumb_create(file, dev, args)` | dumb-buffer alloc (legacy minimal API) | per-driver `Funcs::dumb_create` |
| `drm_gem_dumb_destroy(file, dev, handle)` | dumb-buffer free | per-driver `Funcs::dumb_destroy` |
| `drm_gem_dumb_map_offset(file, dev, handle, &offset)` | dumb-buffer mmap-offset | per-driver `Funcs::dumb_map_offset` |

### compatibility contract

REQ-1: Per-object refcount via `kref` (Rookery: `Refcount`); `drm_gem_object_put` decrements + releases on 0 via per-driver `funcs->free` callback.

REQ-2: Per-FD handle table (xarray) maps userspace-visible u32 handle → kernel `Arc<Object>`; close-fd drops all handles.

REQ-3: Per-object mmap offset allocator via `drm_vma_offset_node`; allocates a unique fake-offset in DEV->vma_offset_manager that, when mmap'd at that offset, faults to per-driver mmap callback.

REQ-4: PRIME export: `DRM_IOCTL_PRIME_HANDLE_TO_FD` looks up local-handle, calls `Object::export_dmabuf` (per-driver overridable) → `dma_buf_export(...)` returns dma-buf-fd; importer can be another DRM device, V4L2, vfio-pci, etc.

REQ-5: PRIME import: `DRM_IOCTL_PRIME_FD_TO_HANDLE` accepts dma-buf-fd, calls `Object::prime_import` (per-driver overridable) → `dma_buf_attach + dma_buf_map_attachment` to backing storage; allocates new GEM object wrapping imported dma-buf; returns local-handle.

REQ-6: Cross-driver buffer share: GPU exports framebuffer, V4L2 imports as capture-target, displayed via DRM atomic-modeset commit (cross-ref `drm-atomic.md`); zero-copy throughout.

REQ-7: Dumb-buffer support (`DRM_IOCTL_MODE_CREATE_DUMB`): minimal portable API for fbdev-style allocation (no per-driver knowledge needed); per-driver `funcs->dumb_create` returns shmem or VRAM-backed object.

REQ-8: Per-object reservation-object (dma_resv) integration: per-object exclusive + shared dma-fence list for GPU↔CPU + GPU↔GPU sync; ww-mutex acquire chain via `drm_gem_lock_reservations` for multi-object atomic operations.

REQ-9: Per-helper backend variants source-compat for in-tree drivers:
- **drm_gem_dma_helper**: physically-contiguous CMA-allocated; for SoC display + simple GPUs.
- **drm_gem_shmem_helper**: shmem-backed system memory; pages allocated lazily on first page-fault.
- **drm_gem_vram_helper**: simple VRAM allocator for ast/mgag200/bochs/cirrus/etc.
- **drm_gem_ttm_helper**: bridge to TTM (cross-ref `drm-ttm.md` future Tier-3) for amdgpu/i915/nouveau/radeon/qxl/virtio-gpu/vmwgfx.

REQ-10: `Object::evict` callback called by TTM-helper or shmem-shrinker under memory pressure; per-driver decides whether to swap-out / migrate / pin / discard.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `gem_no_uaf` | UAF | `Arc<Object>` outlives all per-FD handles + dma-buf attachments + ttm_bo references; release waits for refcount==0. |
| `handle_no_collision` | UNIQUENESS | `xa_alloc` guarantees unique handle id per-FD. |
| `mmap_offset_no_collision` | UNIQUENESS | `drm_vma_offset_node` allocator returns unique offset per-dev. |
| `prime_no_double_export` | INVARIANT | per-object cached `dma_buf` reused across re-export; never two distinct dma-buf objects per GEM object simultaneously. |
| `mmap_no_oob` | OOB | `gem_mmap_obj` validates `vma.vm_end - vma.vm_start <= obj.size`. |

### Layer 2: TLA+

`models/drm/gem_dmabuf.tla` (parent-declared): proves GEM object refcount + dma-buf attach/detach + handle-from-fd + close race — concurrent gem_close + dma-buf release on cross-driver share never produces UAF or leaked refcount.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Object::release` invoked exactly once when refcount reaches 0 | `Arc<Object>` Drop |
| `File::create_gem_handle` post: handle in [1, U32::MAX); xarray lookup returns Some(obj) | `File::create_gem_handle` |
| PRIME export reuses cached dma_buf iff still alive (refcount > 0); else creates fresh one | `File::handle_to_dmabuf_fd` |
| `Object::lock_reservations` post: every obj in `objs` has `obj.resv.lock` held by current task | `Object::lock_reservations` |

### Layer 4: Verus/Creusot functional

`File::handle_to_dmabuf_fd(handle1) → File::dmabuf_fd_to_handle(prime_fd) → handle2` round-trip equivalence: for any GEM object owned by file, exporting then importing produces a handle that refers to the same underlying buffer storage. Encoded as Verus model: `forall file obj. import(export(file, obj)) refers_to obj.storage`.

### hardening

(Inherits row-1 features from `drivers/gpu/drm/00-overview.md` § Hardening.)

gem-core specific reinforcement:

- **Per-FD handle count cap** — bounded at U32::MAX - 1 (avoid handle 0 = invalid); defense against per-FD handle exhaustion.
- **Per-object refcount saturating** — overflow saturates; defense against ref-count-overflow attack.
- **PRIME import LSM mediation** — cross-process dma-buf import goes through file-LSM hook on the source-fd; cross-namespace import additionally validates IOMMU consistency for vfio-imported buffers.
- **Per-object size validated against `dev->driver->dumb_create_max_size`** — defense against userspace allocating absurdly-large GEM objects.
- **Per-object dma_resv exclusive/shared fence list bounded** — per-list cap (default 64 fences); over-cap returns -ENOMEM; defense against fence-list-flood.
- **mmap-offset allocator overflow check** — per-`vma_offset_manager` rb-tree size bounded.
- **Legacy GEM `flink` name globally-visible — DEPRECATED** in Rookery (CONFIG_DRM_LEGACY default-N); modern code uses PRIME instead.
- **Per-FD handle close on FD close guaranteed** — even on process crash, kernel-side cleanup runs in `drm_release` → walks object_handles + drops each.
- **dma_resv ww-mutex deadlock-free via wound-wait** — concurrent multi-object commits converge.

