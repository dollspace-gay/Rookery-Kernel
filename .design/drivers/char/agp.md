# Tier-3: drivers/char/agp/* — AGP graphics aperture + GART (PCI bridge framework, in-kernel API)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/char/agp/backend.c
  - drivers/char/agp/generic.c
  - drivers/char/agp/isoch.c
  - drivers/char/agp/agp.h
  - drivers/char/agp/intel-agp.c
  - drivers/char/agp/intel-gtt.c
  - drivers/char/agp/amd64-agp.c
-->

## Summary

The AGP (Accelerated Graphics Port) subsystem manages the graphics aperture and its GART (Graphics Address Remapping Table) on x86 and selected non-x86 (alpha, parisc, uninorth/PPC) platforms. AGP itself is legacy graphics-bus technology (superseded by PCIe), but the GART hardware lives on in nearly every northbridge / chipset, and several drivers — most importantly Intel integrated graphics via `intel-gtt.c` — still depend on the AGP backend's aperture-management primitives. The backend (`backend.c`) coordinates the active bridge, advertises the driver to `/dev/agpgart`-aware userland (vestigial — see note below), and runs `agp_backend_initialize`/`_cleanup` lifecycle. The generic layer (`generic.c`, ~1400 lines) implements `agp_allocate_memory`/`agp_free_memory`/`agp_bind_memory`/`agp_unbind_memory`, GATT alloc/free, mode-negotiation between bridge and graphics card (`agp_v2_parse_one` / `agp_v3_parse_one`), and per-chipset hook tables (`struct agp_bridge_driver`). Per-vendor drivers (`intel-agp.c`, `amd64-agp.c`, `via-agp.c`, `ati-agp.c`, `nvidia-agp.c`, `sis-agp.c`, `ali-agp.c`, `sworks-agp.c`, `uninorth-agp.c`, `parisc-agp.c`, `alpha-agp.c`) bind a PCI host bridge → register an `agp_bridge_data` → install per-chipset GATT layout and PCI-config tweaks. `intel-gtt.c` is the modern survivor: i915 still calls `intel_gmch_probe` and friends to set up the graphics translation table for integrated Intel GPUs.

Note on `/dev/agpgart` (minor `AGPGART_MINOR = 175`): the `MODULE_ALIAS_MISCDEV` is retained in `backend.c`, but the legacy `frontend.c` chardev with the `AGPIOC_*` ioctl surface (`AGPIOC_ACQUIRE`, `AGPIOC_SETUP`, `AGPIOC_ALLOCATE`, `AGPIOC_BIND`, etc., per `<uapi/linux/agpgart.h>`) is not built in this baseline. AGP today is an in-kernel API only; the uapi header is preserved for ABI documentation and out-of-tree userland that may still link against it. This Tier-3 reflects the as-built (in-kernel) reality and documents the uapi as legacy contract.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct agp_bridge_data` | per-host-bridge AGP state | `drivers::agp::Bridge` |
| `struct agp_bridge_driver` | per-chipset vtable | `drivers::agp::BridgeDriver` |
| `struct agp_memory` | one bound or unbound block of pages | `drivers::agp::Memory` |
| `agp_alloc_bridge()` / `agp_put_bridge(bridge)` | bridge object lifecycle | `Bridge::alloc` / `put` |
| `agp_add_bridge(bridge)` / `agp_remove_bridge(bridge)` | publish/withdraw a bridge | `Subsystem::add_bridge` / `remove_bridge` |
| `agp_backend_initialize(bridge)` / `agp_backend_cleanup(bridge)` | per-bridge init/teardown | `Bridge::initialize` / `cleanup` |
| `agp_find_max()` | scratch-pages heuristic from `maxes_table` | `Subsystem::find_max` |
| `agp_allocate_memory(bridge, page_count, type)` | alloc a GART-bindable chunk | `Memory::allocate` |
| `agp_free_memory(mem)` | release | `Memory::free` |
| `agp_bind_memory(mem, pg_start)` | install into GATT at given page offset | `Memory::bind` |
| `agp_unbind_memory(mem)` | tear out of GATT | `Memory::unbind` |
| `agp_create_user_memory(num_pages)` | unbacked descriptor (userspace-supplied pages) | `Memory::create_user` |
| `agp_create_memory(scratch_pages)` | scratch-backed | `Memory::create_scratch` |
| `agp_generic_alloc_pages` / `agp_generic_destroy_pages` | default per-page backing | `BridgeDriver::generic_alloc_pages` |
| `agp_generic_create_gatt_table(bridge)` / `agp_generic_free_gatt_table(bridge)` | GATT page-table mgmt | `BridgeDriver::gatt_alloc` / `gatt_free` |
| `agp_generic_insert_memory(mem, pg_start, type)` / `agp_generic_remove_memory(mem, pg_start, type)` | per-PTE write/clear | `BridgeDriver::insert` / `remove` |
| `agp_v2_parse_one` / `agp_v3_parse_one` / `agp_collect_device_status` | speed/mode arbitration | `Bridge::negotiate_mode` |
| `agp_3_5_enable(bridge)` | AGP 3.5 isoch enable (`isoch.c`) | `Bridge::enable_isoch` |
| `AGPGART_MINOR` (175) | reserved misc minor for the historical `/dev/agpgart` chardev | `DevIntf::Minor` (legacy) |
| `AGPIOC_INFO` / `_ACQUIRE` / `_RELEASE` / `_SETUP` / `_RESERVE` / `_PROTECT` / `_ALLOCATE` / `_DEALLOCATE` / `_BIND` / `_UNBIND` / `_CHIPSET_FLUSH` | uapi ioctl vocabulary (legacy contract) | `DevIntf::ioctl_*` (legacy) |
| `intel_gmch_probe(pdev, bridge_pdev, bridge)` | i915 entry into intel-gtt | `IntelGtt::probe` |

## Compatibility contract

REQ-1: At most one active `agp_bridge` is the global default; per-system additional bridges link into a list (`agp_bridges`), and clients address each by `struct agp_bridge_data *`.

REQ-2: GATT layout is per-chipset (1-level for legacy AGP, 2-level for AMD64 GART, custom for Intel integrated GTT); the `agp_bridge_driver` vtable hides the difference.

REQ-3: Aperture size is read from a chipset register (e.g. PCI config `AGPGARTLO`/`AGPGARTHI` for legacy AGP) and validated against `maxes_table` page-count ceilings; oversized apertures are clamped.

REQ-4: Page allocation defaults to GFP-DMA32 / per-chipset DMA constraints to ensure GART PTEs can reference the physical page; some chipsets require contiguous backing.

REQ-5: Mode-negotiation reads bridge and card AGPSTAT registers, intersects supported speeds/widths (`AGPx1/x2/x4/x8`) and SBA/FW/over4G capabilities, and programs both sides via AGPCMD.

REQ-6: Per-bridge "key" allocator (`agp_get_key`) returns a stable id for each allocated `agp_memory` so userland (legacy chardev) could refer to it by integer; in the in-kernel API the same key drives debugfs sysfs presentation.

REQ-7: `intel-gtt.c` exports a small private API (`intel_gtt_get`, `intel_gmch_probe`, `intel_gmch_remove`, `intel_gtt_insert_sg_entries`, `intel_gtt_clear_range`) consumed by i915.

REQ-8: AMD64 GART (`amd64-agp.c`) supports apertures up to 2 GB; when CONFIG_GART_IOMMU is also active, the GART is shared as a fallback IOMMU on K8/Family10h CPUs (cross-ref `drivers/iommu/amd-iommu.md`).

REQ-9: `MODULE_ALIAS_MISCDEV(AGPGART_MINOR)` is retained for module autoload by name regardless of whether the chardev is currently built; loading the alias must not depend on a frontend the kernel does not actually export.

## Acceptance Criteria

- [ ] AC-1: On Intel integrated graphics, i915 loads, calls `intel_gmch_probe`, and dmesg reports `Detected <N>K stolen memory` from `intel-gtt`.
- [ ] AC-2: On AMD K8 with CONFIG_GART_IOMMU=y, `dmesg | grep "AGP aperture"` reports the aperture configured by `amd64-agp.c` for shared IOMMU use.
- [ ] AC-3: AGP mode-negotiation produces a non-zero AGPCMD with at least AGPx1 capability on legacy AGP hardware in lab fixtures.
- [ ] AC-4: GATT alloc/free cycles on a real bridge do not leak DMA pages (instrumented test).
- [ ] AC-5: `agp_bind_memory(mem, pg_start)` followed by `agp_unbind_memory(mem)` leaves the GATT entries cleared (TLB-flushed) and the underlying pages still owned by `mem`.
- [ ] AC-6: Concurrent `agp_allocate_memory` from N CPUs serializes on the per-bridge mutex without UAF or duplicated `key`.
- [ ] AC-7: Module unload of an in-use AGP bridge driver is refused (`-EBUSY`) while at least one bound memory block exists.
- [ ] AC-8: `MODULE_ALIAS_MISCDEV(175)` resolves to the `agpgart` module via `modprobe --resolve-alias char-major-10-175` on systems where misc-major maps minor 175.

## Architecture

`Bridge` lives in `drivers::agp::Bridge`:

```
struct Bridge {
  driver: &'static BridgeDriver,
  vm_ops: Option<&'static VmOps>, // legacy mmap support, unused in current build
  dev_private_data: NonNull<()>,
  dev: Arc<PciDev>,
  capndx: u8,                    // PCI capability offset of AGP cap
  mode: u32,                     // negotiated AGPSTAT
  major_version: u8,
  minor_version: u8,
  flags: u32,
  current_size: NonNull<()>,     // points into per-driver aperture_sizes[]
  aperture_size_idx: i32,
  max_memory: usize,             // pages
  current_memory_agp: AtomicUsize,
  type_table: NonNull<()>,
  scratch_page: PhysAddr,
  scratch_page_dma: DmaAddr,
  scratch_page_page: NonNull<Page>,
  gatt_table: NonNull<u32>,      // mapped GATT
  gatt_table_real: NonNull<u32>,
  gatt_bus_addr: DmaAddr,
  gart_bus_addr: PhysAddr,
  apbase_config: u32,
  mapped_table: NonNull<()>,
  mapped_table_size: usize,
  list: ListLink,
  ref_count: AtomicU32,
}

struct Memory {
  key: i32,                      // unique per-bridge identifier
  pages: KVec<NonNull<Page>>,
  num_scratch_pages: u32,
  page_count: u32,
  type_: u32,                    // chipset-specific type code
  physical: DmaAddr,             // for cached-pages mode
  is_bound: AtomicBool,
  is_flushed: bool,
  pg_start: u32,                 // GATT offset when bound
  bridge: Arc<Bridge>,
}
```

Bridge lifecycle:
1. Per-chipset PCI driver matches the host bridge → `agp_alloc_bridge()`.
2. Populates `bridge->driver = &chipset_driver`, reads PCI config (cap offset, aperture base, gart-config), then `agp_add_bridge(bridge)`.
3. `agp_add_bridge` calls `agp_backend_initialize(bridge)`:
   - `agp_find_max()` clamps scratch-pages count against `maxes_table`.
   - `driver->fetch_size(bridge)` → picks aperture-size entry from `aperture_sizes[]` per chipset register.
   - `driver->create_gatt_table(bridge)` → allocates the GATT (or maps a stolen-memory region for integrated GTT).
   - `driver->configure(bridge)` → writes APBASE / APSIZE / GARTLO / GARTHI / GTLB-enable PCI bits.
   - Allocates `scratch_page` for unmapped GATT entries to point at.
4. Per-driver clients (i915 today, X server in legacy lifetimes) hold the bridge and call `agp_allocate_memory` → `agp_bind_memory`.

Mode negotiation `Bridge::negotiate_mode` (`generic.c::agp_v2_parse_one` / `_v3_parse_one`):
1. Read bridge AGPSTAT and card AGPSTAT.
2. Intersect supported AGPx{1,2,4,8} bits and SBA/FW/over4G/RQ-depth capabilities.
3. Apply user-supplied `requested_mode` mask (if any) — caller cannot exceed bridge or card capability.
4. Write AGPCMD on both bridge and card; log `printk(KERN_INFO "%s requested AGPx8 but bridge not capable.")` on degrade.

Memory bind path `agp_bind_memory(mem, pg_start)`:
1. Validate `pg_start + mem->page_count <= bridge->max_memory`.
2. Per-bridge mutex.
3. `driver->insert_memory(mem, pg_start, mem->type_)` writes GATT PTEs page-by-page from `mem->pages[i]` translated through `phys_to_gart` per chipset.
4. `driver->cache_flush()` on chipsets that need it (Intel integrated).
5. `mem->is_bound = true; mem->pg_start = pg_start`.

Intel integrated GTT (`intel-gtt.c`):
- Manages "stolen memory" (DRAM region the BIOS reserves for the IGD).
- Translates DMA-mapped `scatterlist` entries into GTT PTEs via `intel_gtt_insert_sg_entries`.
- `intel_gtt_clear_range(start, count)` zeroes GTT entries back to scratch.
- Used today by `drivers/gpu/drm/i915` for legacy display + gem object pinning on pre-Skylake chipsets.

## Hardening

- Per-bridge mutex serializes `agp_allocate_memory`, `agp_bind_memory`, and `agp_unbind_memory`; concurrent callers cannot corrupt GATT mid-write.
- GATT PTEs are written before the chipset TLB flush; ordering is enforced via `wmb()` + chipset-specific `cache_flush`.
- `mem->key` is allocated from a bridge-local IDA; collisions impossible.
- Module-unload path checks per-bridge bound-memory count; refuses unload while clients hold mappings.
- Aperture base is validated against PCI BAR-claimed range; chipset drivers refuse to register a bridge whose aperture overlaps a non-AGP BAR.
- `scratch_page` is a dedicated zero page; unbound GATT entries point at it so stray device reads of unmapped pages do not see other kernel memory.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `agp_bridge_data`, `agp_memory`, and per-bridge GATT shadow buffers; legacy userland buffers (if a future `/dev/agpgart` re-emerges) strictly bounded against `agp_info`/`agp_allocate`/`agp_bind` sizes.
- **PAX_KERNEXEC** — `agp_bridge_driver` vtables, generic insert/remove helpers, and per-chipset PCI-config writers live in `__ro_after_init` text; no runtime patching of bridge ops.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `agp_bind_memory`, `agp_allocate_memory`, GATT insert/remove, and mode-negotiation entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `agp_bridge_data.ref_count` and on per-`agp_memory` reference tracking; overflow trap defeats UAF on race between client release and bridge unload.
- **PAX_MEMORY_SANITIZE** — zero-on-free for GATT-backing pages, `agp_memory.pages[]` arrays, and the bridge scratch page so prior DMA-visible contents cannot bleed across bind cycles.
- **PAX_UDEREF** — SMAP/PAN enforced on any (legacy) ioctl entry; `agp_setup`/`agp_allocate`/`agp_bind`/`agp_region` copied only through canonical helpers if the chardev ever re-enables.
- **PAX_RAP / kCFI** — `agp_bridge_driver` vtable, `vm_operations_struct`, and intel-gtt private ops are typed indirect calls; misrouted chipset plugin traps rather than executing wrong handler.
- **GRKERNSEC_HIDESYM** — suppress `%p` of `agp_bridge_data` / `agp_memory` / GATT base in `/proc/driver/agp*` and tracepoints; gate behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict bridge-discovery, aperture-base, GATT-bus-address, and mode-negotiation banners to CAP_SYSLOG so attackers cannot probe DMA-capable apertures via dmesg.
- **/dev/agpgart CAP_SYS_RAWIO + lockdown integrity** — if the legacy chardev is ever re-built, `AGPIOC_ACQUIRE`/`_SETUP`/`_BIND` are DMA-bypass primitives (they program a translation table the GPU may use to alias arbitrary host pages) and must require CAP_SYS_RAWIO and refuse when `security_locked_down(LOCKDOWN_DEV_MEM)` is active.
- **Aperture mmap NX-bit** — pages mapped from the AGP aperture into userland (legacy `agp_mmap`) get the NX bit so attacker-controlled GART aliasing cannot be turned directly into executable user memory.
- **GART entries bounded** — every `agp_bind_memory` call validates `pg_start + page_count` against `bridge->max_memory`; reject negative `pg_start`, oversized blocks, and overlapping bindings.
- **AGP-bridge match table RO** — the PCI ID tables in each `*-agp.c` driver marked `__ro_after_init`; defense against device-ID spoofing via writable data.
- **Per-page DMA-mask check** — every `agp_memory.pages[i]` validated against `bridge->dev->dma_mask` before GATT install; refuse pages above the chipset addressable range.
- **Stolen-memory bounds (Intel GTT)** — `intel-gtt.c` validates BIOS-reported stolen-base/size against the e820 reserved-memory map; refuse to bind GTT entries pointing into non-stolen kernel or user RAM.

Rationale: GART hardware is a DMA-capable address translation table — the same threat model as an IOMMU but generally without the validation rigor. A bug that lets a graphics device bind a GART entry pointing at kernel `.text` or another process's page is a complete machine compromise. CAP_SYS_RAWIO on bind paths, lockdown integrity gating on DMA-bypass ioctls, NX on aperture mmaps, page-count and DMA-mask validation on every bind, refcount saturation across bridge teardown, and kCFI on per-chipset vtables turn AGP from "ancient legacy with implicit trust" into a bounded structural boundary suitable for shared environments.

## Open Questions

- Should the in-kernel API (intel-gtt et al.) carry an iommufd-style ownership claim so a future user-space DRM driver cannot quietly re-acquire a GART being driven by i915?
- Re-introducing the legacy `/dev/agpgart` chardev for nostalgia/test would re-open the AGPIOC ioctl surface — is that ever justified, or should the alias be removed entirely?

## Out of Scope

- Per-chipset PCI-config quirks (covered in `intel-agp.md`, `amd64-agp.md`, etc. — future Tier-3 candidates if their complexity warrants)
- DRM/i915 consumer side (covered in `drivers/gpu/drm/i915/*.md` future Tier-3)
- Alpha / parisc / PPC AGP backends (architecture-specific Tier-3 if revived)
- AGP isochronous transfers (`isoch.c`) deep dive
- Implementation code
