# Tier-3: drivers/uio/uio.c — Userspace I/O framework (/dev/uioN, mmap regions, IRQ handler delegation)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/uio/uio.c
  - include/linux/uio_driver.h
-->

## Summary

UIO (Userspace I/O) is the kernel framework that lets a userspace process drive a hardware device by mapping its MMIO regions + receiving its interrupts — without writing a full in-kernel driver. The kernel still owns the device probe + IRQ vector + memory regions; userspace owns the per-register protocol. Each registered UIO device exposes:

- A char device `/dev/uio<N>` (major `uio_major`, minor allocated from `uio_idr`).
- `read(/dev/uioN, &count, 4)` → blocks until next IRQ; returns monotonically-increasing event count.
- `write(/dev/uioN, &enable, 4)` → enables/disables the device's IRQ via the in-kernel `irqcontrol` callback.
- `mmap(/dev/uioN, ..., offset = N*PAGE_SIZE)` → maps memory region `info->mem[N]` (PHYS/LOGICAL/VIRTUAL/DMA_COHERENT/IOVA) into userspace.
- `poll(/dev/uioN, POLLIN)` → wakes on event.
- `/sys/class/uio/uio<N>/maps/map<N>/{name,addr,size,offset}` — per-region metadata.
- `/sys/class/uio/uio<N>/portio/port<N>/...` — per-port-region metadata (x86 IO ports).

This Tier-3 covers `drivers/uio/uio.c` (~1150 lines) + `include/linux/uio_driver.h`. The in-tree consumer drivers (`uio_pci_generic.c`, `uio_hv_generic.c`, `uio_pdrv_genirq.c`, `uio_dmem_genirq.c`, `uio_dfl.c`, `uio_aec.c`, `uio_cif.c`, `uio_mf624.c`, `uio_netx.c`, `uio_sercos3.c`, `uio_fsl_elbc_gpcm.c`, `uio_pci_generic_sva.c`) are out of scope here.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct uio_device` | per-device control block: dev, minor, event counter, async_queue, wait_queue, info, info_lock, map_dir, portio_dir | `uio::Device` |
| `struct uio_info` | per-driver vtable + region descriptors: name, version, mem[MAX_UIO_MAPS], port[MAX_UIO_PORT_REGIONS], irq, handler, mmap_prepare, open, release, irqcontrol | `uio::Info` |
| `struct uio_mem` | per-region descriptor: name, addr, dma_addr, offs, size, memtype, internal_addr, dma_device, map | `uio::Mem` |
| `struct uio_port` | per-port-region descriptor: name, start, size, porttype, portio | `uio::Port` |
| `struct uio_listener` | per-`open` state: dev pointer + event_count snapshot | `uio::Listener` |
| `MAX_UIO_MAPS = 5` | per-device map cap | `uio::MAX_MAPS` |
| `MAX_UIO_PORT_REGIONS = 5` | per-device port cap | `uio::MAX_PORTS` |
| `UIO_MAX_DEVICES = (1U << MINORBITS)` | global device cap (1 << 20) | `uio::MAX_DEVICES` |
| `__uio_register_device(owner, parent, info)` / `uio_unregister_device(info)` | per-driver register/unregister | `Device::register` / `_unregister` |
| `__devm_uio_register_device(owner, parent, info)` | devres-managed register | `Device::register_devm` |
| `uio_event_notify(info)` | post-IRQ event wake-up (called from driver IRQ context) | `Device::event_notify` |
| `uio_interrupt(irq, dev_id)` | top-half: delegates to `info->handler`, then `uio_event_notify` | `Device::interrupt` |
| `uio_open(inode, filep)` / `_release(...)` | char-device open/close + per-listener alloc | `Device::open` / `_release` |
| `uio_read(filep, buf, count, ppos)` | block-on-event read returning u32 event count | `Device::read` |
| `uio_write(filep, buf, count, ppos)` | irq_on writes -> `info->irqcontrol(info, irq_on)` | `Device::write` |
| `uio_mmap(filep, vma)` | dispatches per memtype: physical / logical / dma_coherent / iova | `Device::mmap` |
| `uio_mmap_physical(vma)` | `remap_pfn_range` for UIO_MEM_PHYS / UIO_MEM_IOVA | `Device::mmap_physical` |
| `uio_mmap_logical(vma)` | per-fault `virt_to_page`/`vmalloc_to_page` for UIO_MEM_LOGICAL / UIO_MEM_VIRTUAL | `Device::mmap_logical` |
| `uio_mmap_dma_coherent(vma)` | `dma_mmap_coherent` for UIO_MEM_DMA_COHERENT | `Device::mmap_dma_coherent` |
| `uio_poll(filep, wait)` / `_fasync(fd, filep, on)` | poll + SIGIO | `Device::poll` / `_fasync` |
| `static DEFINE_IDR(uio_idr)` | minor allocator | `Subsystem::idr` |
| `static DEFINE_MUTEX(minor_lock)` | guards idr | `Subsystem::minor_lock` |
| `static struct cdev *uio_cdev` | char-device cdev | `Subsystem::cdev` |
| `static int uio_major` | dynamically-allocated char-device major | `Subsystem::major` |

## Compatibility contract

REQ-1: `uio_init` (module init) calls `uio_major_init` which calls `alloc_chrdev_region(&uio_dev, 0, UIO_MAX_DEVICES, "uio")` and `cdev_alloc()` + `cdev_add(uio_cdev, uio_dev, UIO_MAX_DEVICES)`.

REQ-2: `class_register(&uio_class)` registers `/sys/class/uio/`. Class release callback frees per-`uio_device` data.

REQ-3: `__uio_register_device(owner, parent, info)` MUST be called with `info->name`, `info->version`, and at least one of `info->mem[].size != 0` or `info->port[].size != 0` or `info->irq != UIO_IRQ_NONE`.

REQ-4: Minor allocation: `idr_alloc(&uio_idr, idev, 0, UIO_MAX_DEVICES, GFP_KERNEL)` under `minor_lock`; failure rolls back partially-allocated state.

REQ-5: Per-device sysfs: `/sys/class/uio/uio<N>/{name,version,event}` + per-map subdirs `maps/mapN/{name,addr,size,offset}` + per-port subdirs `portio/portN/{name,start,size,porttype}`.

REQ-6: IRQ registration: if `info->irq != UIO_IRQ_NONE && info->irq != UIO_IRQ_CUSTOM`, framework calls `request_irq(info->irq, uio_interrupt, info->irq_flags, info->name, idev)` + provides default top-half that invokes `info->handler` then `uio_event_notify`.

REQ-7: `info->irq == UIO_IRQ_CUSTOM`: driver manages its own IRQ; framework relies on driver calling `uio_event_notify(info)` from its handler.

REQ-8: `uio_event_notify`: `atomic_inc(&idev->event)` + `wake_up_interruptible(&idev->wait)` + `kill_fasync(&idev->async_queue, SIGIO, POLL_IN)`.

REQ-9: `uio_read(buf, count)`: validates `count == 4`; loop while `event_count == listener->event_count`; on wake, copy current `event` to userspace; non-blocking returns `-EAGAIN`.

REQ-10: `uio_write(buf, count)`: validates `count == 4`; reads s32 `irq_on`; requires `info->irq` non-zero + `info->irqcontrol` non-NULL; invokes `info->irqcontrol(info, irq_on)`.

REQ-11: `uio_mmap`: `vma->vm_pgoff` is the map index (0..MAX_UIO_MAPS-1); `vma_pages(vma)` MUST NOT exceed `actual_pages = ceil((addr & ~PAGE_MASK + size) / PAGE_SIZE)`.

REQ-12: `uio_mmap_physical`: `remap_pfn_range(vma, vma->vm_start, mem->addr >> PAGE_SHIFT, len, prot)`; UIO_MEM_PHYS uses `pgprot_noncached`; UIO_MEM_IOVA uses default cached prot.

REQ-13: `uio_mmap_logical`: installs `uio_logical_vm_ops.fault` (`uio_vma_fault`) which does on-fault `virt_to_page` (UIO_MEM_LOGICAL) or `vmalloc_to_page` (UIO_MEM_VIRTUAL); `vm_flags = VM_DONTEXPAND | VM_DONTDUMP`.

REQ-14: `uio_mmap_dma_coherent`: requires `mem->dma_device != NULL` and `(mem->addr | mem->dma_addr) & PAGE_MASK == 0`; framework `dev_warn` discourages this memtype.

REQ-15: `mmap_prepare` (newer hook): if `info->mmap_prepare` is set, framework calls it first; allows driver to set up `vm_area_desc` programmatically (used by DFL UIO).

REQ-16: `uio_open` per `open`: `idr_find_locked(&uio_idr, iminor(inode))` resolves device; allocates `struct uio_listener`; if `info->open != NULL` and `idev->info != NULL`, calls `info->open(info, inode)`.

REQ-17: `uio_release`: calls `info->release(info, inode)` if set; frees listener; if `info` has been unregistered while a listener is open, `idev->info = NULL` and ops return -EINVAL.

REQ-18: `uio_unregister_device`: under `idev->info_lock` set `idev->info = NULL`, free IRQ if framework-owned, then `device_unregister(&idev->dev)` + `idr_remove(&uio_idr, minor)`.

## Acceptance Criteria

- [ ] AC-1: `modprobe uio_pci_generic; setpci -s <pci> ...; /dev/uio0` appears with correct `/sys/class/uio/uio0/maps/map0/{name,addr,size}` for BAR0.
- [ ] AC-2: Userspace `open("/dev/uio0", O_RDWR)` + `mmap(NULL, size, RW, MAP_SHARED, fd, 0)` succeeds + maps BAR0 MMIO.
- [ ] AC-3: Userspace blocking `read(fd, &count, 4)` blocks until device IRQ; returns event count == previous + 1.
- [ ] AC-4: Userspace `write(fd, &irq_on, 4)` with `irq_on=1` re-enables IRQ via in-kernel `irqcontrol`.
- [ ] AC-5: `poll(fd, POLLIN, timeout)` wakes within timeout on IRQ.
- [ ] AC-6: `fcntl(fd, F_SETFL, O_NONBLOCK); read(fd, &count, 4)` returns `-EAGAIN` when no event pending.
- [ ] AC-7: `uio_mmap` of UIO_MEM_PHYS region is non-cached (verified via `mtrr`/`pat`).
- [ ] AC-8: Driver unbind while `/dev/uioN` is open: subsequent `read`/`write`/`mmap` return `-EINVAL`; `release` cleans up.
- [ ] AC-9: Per-region addr/size sysfs reads match `info->mem[N]` provided at register.
- [ ] AC-10: `kselftest tools/testing/selftests/uio/` passes (when present).
- [ ] AC-11: Hyper-V netvsc fast-path via `uio_hv_generic` and DPDK via `uio_pci_generic` both functional.

## Architecture

### Subsystem state

Globals: `uio_major` (alloc_chrdev_region), `uio_cdev`, `uio_idr` (minor → device map), `minor_lock` (guards idr), `uio_fops`, and the `uio_class` (`.name = "uio"`, `.dev_release = uio_dev_release`).

### Per-device state

`struct uio_device` carries: `owner` (driver module), `dev` (under `/sys/class/uio/uioN`), `minor`, `event` (atomic monotonic counter), `async_queue`, `wait` (waitqueue), `info` (driver-provided, cleared on unregister), `info_lock` (protects `info` pointer), `map_dir`, `portio_dir`.

`struct uio_info` carries: `uio_dev` back-pointer, `name`, `version`, `mem[MAX_UIO_MAPS]`, `port[MAX_UIO_PORT_REGIONS]`, `irq` (UIO_IRQ_NONE / UIO_IRQ_CUSTOM / real), `irq_flags`, `priv`, and the vtable (`handler`, `mmap_prepare`, `open`, `release`, `irqcontrol`).

### Register flow (`__uio_register_device`)

1. Validate `info->name`, `info->version`, `info->mem[]` / `info->port[]` / `info->irq` not all zero.
2. `idev = kzalloc(sizeof(*idev))` + back-pointer + lock init.
3. Under `minor_lock`: `idr_alloc(&uio_idr, idev, 0, UIO_MAX_DEVICES, GFP_KERNEL)` → `idev->minor`.
4. `idev->dev.devt = MKDEV(uio_major, idev->minor)` + `dev_set_name(&idev->dev, "uio%d", idev->minor)` + `device_register(&idev->dev)`.
5. `uio_dev_add_attributes(idev)` — build per-map and per-port sysfs subdirs.
6. If `info->irq && info->irq != UIO_IRQ_CUSTOM`: `request_irq(info->irq, uio_interrupt, info->irq_flags, info->name, idev)`.
7. `info->uio_dev = idev`.

### Read/event path

`uio_read(fd, buf, count)`: validates `count == sizeof(s32)`; loops `add_wait_queue(&idev.wait)` + check `idev.info` under `info_lock` (return `-EINVAL` if NULL) + `TASK_INTERRUPTIBLE` + compare `atomic_read(&idev.event)` against `listener.event_count`; on mismatch wake, copy current event count to userspace and update listener; `O_NONBLOCK` returns `-EAGAIN`; signal pending returns `-ERESTARTSYS`.

### Write/IRQ control path

`uio_write(fd, buf, count)`: validates `count == sizeof(s32)`; `copy_from_user` an s32 `irq_on`; under `info_lock` reject if `info == NULL` (`-EINVAL`), `info.irq == 0` (`-EIO`), `info.irqcontrol == NULL` (`-ENOSYS`); delegates to `info.irqcontrol(info, irq_on)`.

### IRQ delegation (`uio_interrupt`)

`uio_interrupt(irq, dev_id)` invokes `info.handler(irq, info)` and, if result is `IRQ_HANDLED`, calls `uio_event_notify(info)` which atomic-incs `event`, wakes the waitqueue, and `kill_fasync(SIGIO, POLL_IN)`. Driver-side `handler` does the bare minimum to silence the device (read interrupt-status, ack); the framework owns event delivery.

### mmap dispatch

`uio_mmap(filep, vma)`: stash `vma.vm_private_data = idev`; under `info_lock` resolve `mi = uio_find_mem_index(vma)`; refuse if `vma_pages(vma) > ceil((mem.addr & ~PAGE_MASK + mem.size) / PAGE_SIZE)`; if `info.mmap_prepare` set, delegate via `__compat_vma_mmap`; otherwise dispatch on memtype: `UIO_MEM_PHYS`/`UIO_MEM_IOVA` → `uio_mmap_physical`, `UIO_MEM_LOGICAL`/`UIO_MEM_VIRTUAL` → `uio_mmap_logical`, `UIO_MEM_DMA_COHERENT` → `uio_mmap_dma_coherent`.

`uio_mmap_physical` uses `remap_pfn_range(vma, vma.vm_start, mem.addr >> PAGE_SHIFT, len, vma.vm_page_prot)`; for UIO_MEM_PHYS applies `pgprot_noncached` first. `uio_mmap_logical` installs `uio_logical_vm_ops.fault = uio_vma_fault` + `VM_DONTEXPAND | VM_DONTDUMP`; per-page fault: `vmf->pgoff` → mem index → kernel virt addr → `virt_to_page` / `vmalloc_to_page` → `get_page` → `vmf->page`. `uio_mmap_dma_coherent` calls `dma_mmap_coherent(mem.dma_device, vma, kaddr, mem.dma_addr, len)` with `vma.vm_pgoff` temporarily cleared (UIO uses pgoff as map index but `dma_mmap_coherent` expects 0).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mmap_no_oob` | OOB | `requested_pages ≤ actual_pages`; refuse VMA bigger than region |
| `read_buf_no_oob` | OOB | `count == sizeof(s32)` validated before `copy_to_user` |
| `write_buf_no_oob` | OOB | `count == sizeof(s32)` validated before `copy_from_user` |
| `info_pointer_no_uaf` | UAF | every `idev.info` deref under `info_lock`; unregister sets to NULL |
| `idr_no_collision` | UNIQUENESS | minor unique within `uio_idr`; `idr_alloc` semantics + `minor_lock` |
| `vma_fault_page_valid` | UAF | `uio_vma_fault` returns VM_FAULT_SIGBUS when mem index invalid or info NULL |
| `irq_dispatch_after_handler` | ORDERING | `uio_event_notify` invoked only after `info.handler` returned IRQ_HANDLED |

### Layer 2: TLA+

`models/uio/event_count.tla` (parent-declared): models N userspace readers + driver IRQ + concurrent open/close. Property: every IRQ produces exactly one `event` increment + wakeup; no reader returns the same event_count twice; no event is dropped while ≥ 1 listener is registered.

`models/uio/unregister_race.tla`: models `uio_unregister_device` racing with active `read`/`write`/`mmap`. Property: post-unregister ops return -EINVAL cleanly; no UAF on `idev.info`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Device::read` post: returned event_count == `atomic_read(&idev.event)` at the moment of wake | `Device::read` |
| `Device::write` post: `info.irqcontrol` called iff `info.irq != 0 ∧ info.irqcontrol != NULL` | `Device::write` |
| `Device::mmap` post: VMA installed iff `mem.size > 0 ∧ vma_pages ≤ actual_pages` | `Device::mmap` |
| `Device::register` post: minor allocated ∧ char-dev visible ∧ IRQ requested (if framework-owned) | `Device::register` |
| `Device::unregister` post: `info` cleared ∧ IRQ freed ∧ minor returned to `uio_idr` | `Device::unregister` |

### Layer 4: Verus/Creusot functional

End-to-end DPDK-style userspace driver: kernel `uio_pci_generic` probe → UIO register → /dev/uio0 visible → userspace open + mmap BAR + setup IRQ in disabled state → device IRQ fires → kernel `uio_interrupt` runs in-kernel disable-IRQ-at-source via `info.handler` → `uio_event_notify` → userspace `read` returns event count → userspace processes packet → userspace `write(irq_on=1)` re-enables IRQ. Encoded as a Verus refinement: every IRQ produces exactly one observable userspace event count delta.

## Hardening

(Inherits from `drivers/00-overview.md` § Hardening; UIO is privileged because it exposes raw MMIO + IRQs to userspace.)

UIO-specific reinforcement:

- **`UIO_MAX_DEVICES = 1 << MINORBITS`** — bounded; defense against minor-space exhaustion.
- **`MAX_UIO_MAPS = 5` + `MAX_UIO_PORT_REGIONS = 5`** — bounded; defense against allocation explosion.
- **`info_lock` per device** — all `info` derefs serialized; defense against UAF across unregister.
- **`idev.info = NULL` on unregister** — fileops return `-EINVAL` thereafter.
- **`remap_pfn_range` only on per-region addr/size** — defense against mapping arbitrary PFNs.
- **UIO_MEM_PHYS gets `pgprot_noncached`** — defense against caching MMIO.
- **`VM_DONTEXPAND | VM_DONTDUMP` on logical mappings** — defense against mremap + coredump leakage.
- **`uio_find_mem_index` rejects out-of-range `vm_pgoff`** — defense against negative or oversize index.
- **IRQ handler delegation pattern** — framework owns top-half registration; driver provides minimal silencer; userspace owns protocol — defense against userspace blocking IRQ delivery.
- **`dev_warn` on UIO_MEM_DMA_COHERENT** — discouraged memtype because of IOMMU + lifetime issues.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `struct uio_device`, `uio_listener`, `uio_map`, `uio_portio`; `read`/`write` strictly `sizeof(s32)`-bounded; `copy_to_user`/`copy_from_user` reject any other length.
- **PAX_KERNEXEC** — `uio.c` text in W^X kernel text; `uio_fops`, `uio_logical_vm_ops`, `uio_physical_vm_ops`, `uio_class.dev_release`, the sysfs `kobj_type` show/store dispatch, and the IRQ default handler all `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `uio_read`, `uio_write`, `uio_mmap`, `uio_interrupt`, `uio_vma_fault`, and `__uio_register_device`.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-device kref + per-listener refcount; `atomic_t event` migrated to saturating counter to prevent wrap-around DoS.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `uio_device`, `uio_listener`, `uio_map`, `uio_portio`, and per-region descriptor caches so stale BAR addresses, DMA handles, and IRQ vectors cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on `uio_read`/`uio_write` `copy_*_user` boundaries and on `uio_mmap` `vma` field reads; `vm_pgoff` validated before use.
- **PAX_RAP / kCFI** — `uio_fops`, `uio_logical_vm_ops`, `uio_physical_vm_ops`, `info->handler`, `info->mmap_prepare`, `info->open`, `info->release`, `info->irqcontrol`, and `uio_class.dev_release` marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-device `info` pointer / BAR-addr disclosure behind CAP_SYSLOG; suppress `%p` in /sys/class/uio/* attributes.
- **GRKERNSEC_DMESG** — restrict UIO register-fail, IRQ-request-fail, mmap-reject, and unregister banners to CAP_SYSLOG; per-event tracepoints gated.
- **`/dev/uio*` CAP_SYS_RAWIO + lockdown-mode integrity gate** — `uio_open` checks `capable(CAP_SYS_RAWIO)` AND refuses when `kernel_is_locked_down(LOCKDOWN_DEV_MEM)`; defense against unprivileged userspace MMIO ownership; matches the `/dev/mem` / `/dev/kmem` lockdown story.
- **mmap PAX_KERNEXEC NX-bit** — `vma->vm_flags & VM_EXEC` cleared by `uio_mmap` before installing VM ops; refuse `PROT_EXEC` mappings; defense against userspace executing from MMIO/DMA memory.
- **IRQ ack writes bounded** — `info->irqcontrol(info, irq_on)` argument validated: only `0`/`1` accepted; defense against userspace passing arbitrary `s32` that a careless driver writes raw to a register.
- **Name allowlist for /sys/class/uio** — `info->name` validated against `[a-zA-Z0-9._-]{1,32}` regex at register time; sysfs node name sanitized; defense against driver supplying attacker-controlled sysfs path.
- **DMA-coherent memtype refused under lockdown** — `uio_mmap_dma_coherent` returns -EPERM under `LOCKDOWN_DEV_MEM`; defense against userspace direct-DMA without IOMMU isolation.
- **`info->mem[i].size == 0` is end-of-list marker** — strict validation refuses non-zero entries after a zero-size entry; defense against driver-supplied malformed region table.
- **No mremap on UIO VMAs** — `VM_DONTEXPAND` set by `uio_mmap_logical`; physical mappings naturally non-expandable via `remap_pfn_range`; defense against userspace expanding the mapping past region bounds.

Rationale: UIO is intentionally a hole in the "kernel owns hardware" model — DPDK, SPDK, FPGA frameworks, Hyper-V netvsc fast-path, and a long tail of industrial controllers all run with `/dev/uio*` mapped into a userspace process. That hole MUST be gated: CAP_SYS_RAWIO + lockdown-mode integrity, NX-bit on mappings, validated IRQ-control arguments, sanitized sysfs names, refused DMA-coherent under lockdown, and kCFI on the per-driver vtables collapse the attack surface from "any DPDK app is potentially root-equivalent" into a verified consent boundary with hardware-level isolation guarantees.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-driver UIO bus glue (`uio_pci_generic.c`, `uio_hv_generic.c`, `uio_pdrv_genirq.c`, `uio_dmem_genirq.c`, `uio_dfl.c`, `uio_aec.c`, `uio_cif.c`, `uio_mf624.c`, `uio_netx.c`, `uio_sercos3.c`, `uio_fsl_elbc_gpcm.c`, `uio_pci_generic_sva.c`): future Tier-3 docs per driver.
- VFIO (`drivers/vfio/`): the IOMMU-grouped alternative to UIO; separate subsystem.
- DPDK / SPDK userspace driver design: out of kernel scope.
- Implementation code.
