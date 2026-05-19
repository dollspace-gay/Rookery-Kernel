# Tier-3: drivers/remoteproc/{remoteproc_core,remoteproc_elf_loader}.c — Remoteproc lifecycle + ELF loader + carveouts

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/remoteproc/remoteproc-core.md
upstream-paths:
  - drivers/remoteproc/remoteproc_core.c
  - drivers/remoteproc/remoteproc_elf_loader.c
  - drivers/remoteproc/remoteproc_elf_helpers.h
  - drivers/remoteproc/remoteproc_internal.h
  - drivers/remoteproc/remoteproc_virtio.c
  - drivers/remoteproc/remoteproc_sysfs.c
  - drivers/remoteproc/remoteproc_cdev.c
  - include/linux/remoteproc.h
-->

## Summary

Remoteproc — the kernel framework that owns the lifecycle of secondary processors on heterogeneous SoCs (Cortex-M co-processors, DSPs, sensor hubs, modem subsystems, security enclaves) plus their firmware loading, resource-table parsing, carveout allocation, optional IOMMU programming, optional virtio-over-shared-memory transport (vdev), and crash-recovery state machine. Boards plug in a `rproc_ops` (Qualcomm Q6, TI K3, STM32, NXP iMX, Xilinx R5, etc.) and remoteproc handles ELF parse + segment load + boot/shutdown/recover + sysfs/cdev exposure.

This Tier-3 covers `remoteproc_core.c` (~2800 lines: rproc lifecycle, carveout mgmt, resource-table handlers, recovery, panic notifier) + `remoteproc_elf_loader.c` (~400 lines: ELF32/ELF64 sanity-check + segment loader).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rproc` | per-rproc state: ops, firmware name, state, carveouts, mappings, traces, vdevs, recovery work | `drivers::remoteproc::Rproc` |
| `struct rproc_ops` | per-platform vtable: prepare, unprepare, start, stop, attach, kick, da_to_va, parse_fw, find_loaded_rsc_table, sanity_check, get_boot_addr, load, panic | `drivers::remoteproc::Ops` |
| `struct rproc_mem_entry` | per-carveout: da, len, dma, va, name, alloc/release ops | `drivers::remoteproc::MemEntry` |
| `rproc_boot(rproc)` | reference-counted bring-up: load fw → parse rsc → alloc carveouts → start | `Rproc::boot` |
| `rproc_shutdown(rproc)` | reference-counted tear-down: stop → free carveouts → release rsc | `Rproc::shutdown` |
| `rproc_fw_boot(rproc, fw)` | one-time-boot helper: sanity → load → resource handlers → start | `Rproc::fw_boot` |
| `rproc_load_segments(rproc, fw)` | per-PT_LOAD segment copy into carveout-mapped VA | `Rproc::load_segments` |
| `rproc_elf_sanity_check(rproc, fw)` | ELF32/ELF64 header + phoff/shoff/phnum validation | `Elf::sanity_check` |
| `rproc_elf_get_boot_addr(rproc, fw)` | entry-point address | `Elf::boot_addr` |
| `rproc_handle_resources(rproc, handlers)` | per-rsc dispatch (carveout, devmem, trace, vdev, last) | `Rproc::handle_resources` |
| `rproc_alloc_carveout(rproc, mem)` / `_release_carveout(...)` | dma_alloc_coherent-backed carveout + optional IOMMU map | `Rproc::alloc_carveout` |
| `rproc_iommu_fault(domain, dev, iova, flags, token)` | IOMMU fault callback → triggers `rproc_report_crash(MMUFAULT)` | `Rproc::iommu_fault` |
| `rproc_report_crash(rproc, type)` | transition to RPROC_CRASHED, schedule recovery | `Rproc::report_crash` |
| `rproc_boot_recovery(rproc)` | recovery worker: shutdown + boot under workqueue | `Rproc::boot_recovery` |
| `rproc_alloc(dev, name, ops, fw_name, len)` / `rproc_add(rproc)` | alloc + register; `rproc_del`, `rproc_free` | `Rproc::{alloc,add,del,free}` |
| `rproc_da_to_va(rproc, da, len, is_iomem)` | translate device-address to kernel-VA via carveout list / ops | `Rproc::da_to_va` |
| sysfs (`remoteproc_sysfs.c`) | per-rproc `state`, `firmware`, `name`, `coredump`, `recovery` attrs | `Rproc::sysfs` |
| cdev (`remoteproc_cdev.c`) | per-rproc `/dev/remoteproc<N>` for control: boot/stop, FW reload | `Rproc::cdev` |
| `remoteproc_virtio.c` | vdev resource-entry → virtio_device over shared carveout + rpmsg backend | cross-ref `drivers/rpmsg/rpmsg-core.md` |

## Compatibility contract

REQ-1: Per-rproc state machine: RPROC_OFFLINE → RPROC_SUSPENDED → RPROC_RUNNING → RPROC_CRASHED → RPROC_DELETED; plus RPROC_DETACHED for attach-on-boot (early-attach when bootloader pre-booted the rproc).

REQ-2: ELF loader supports ELF32 + ELF64, host-endianness only, PT_LOAD segments with `filesz ≤ memsz`, segment offsets validated against fw->size (no out-of-band read).

REQ-3: Resource table format per `include/linux/remoteproc.h`: RSC_CARVEOUT, RSC_DEVMEM, RSC_TRACE, RSC_VDEV, RSC_LAST + RSC_VENDOR (per-platform). Per-handler validates entry size before dispatch.

REQ-4: Per-carveout DMA-coherent allocation via `dma_alloc_coherent` on `rproc->dev.parent`; if `rproc->has_iommu`, additional `iommu_map` into the per-rproc iommu_domain.

REQ-5: Per-rproc IOMMU domain (cross-ref `iommu-core.md`): `iommu_paging_domain_alloc(dev)` + `iommu_attach_device`; on fault → `rproc_iommu_fault` reports crash.

REQ-6: Firmware naming + loading via `firmware_class.c`: `request_firmware(&fw, rproc->firmware, &rproc->dev)`; supports per-rproc firmware-override via sysfs `firmware` attr.

REQ-7: Recovery policy via sysfs `recovery` attr: `enabled`, `disabled`, `recovery-disabled`. Per-recovery cycle goes through `rproc_recovery_wq` workqueue.

REQ-8: Panic notifier `rproc_panic_nb` invokes per-rproc `ops->panic()` so co-processors can flush state on host panic (for post-mortem analysis).

REQ-9: Per-rproc virtio device(s) instantiated from vdev resource entries; vring carveouts allocated per-vdev; transport via shared carveout + kicks via `rproc->ops->kick`.

REQ-10: Per-rproc tracebuffer entries (RSC_TRACE) exposed via debugfs `/sys/kernel/debug/remoteproc/remoteproc<N>/trace<i>` as binary blob backed by carveout.

REQ-11: Coredump support (`remoteproc_coredump.c`) — per-crash ELF coredump written to `/sys/class/devcoredump/devcd<N>/data` with per-segment dumps configurable as inline / mmapped / disabled.

REQ-12: Per-rproc auto-boot via `rproc->auto_boot` flag set at probe; `rproc_add` boots immediately if set (otherwise userland triggers via sysfs `state`).

## Acceptance Criteria

- [ ] AC-1: `cat /sys/class/remoteproc/remoteproc0/state` shows `offline` after probe, `running` after `echo start`.
- [ ] AC-2: Firmware reload: `echo new_fw.elf > firmware`; subsequent boot loads new ELF.
- [ ] AC-3: Force crash via fault injection → state transitions to `crashed`; recovery workqueue restores `running` within bounded time.
- [ ] AC-4: ELF sanity-check rejects: truncated header, mismatched endianness, phnum=0, `filesz > memsz`, offset overflow.
- [ ] AC-5: IOMMU-backed rproc: fault on bad device-address triggers `rproc_iommu_fault` → crash → recovery; no host kernel panic.
- [ ] AC-6: vdev path: `rpmsg_char` `/dev/rpmsg_ctrl0` becomes available after rproc boot of an rpmsg-capable firmware.
- [ ] AC-7: Coredump: forced crash produces `/sys/class/devcoredump/devcdN/data` consumable by elf-readers.
- [ ] AC-8: Panic notifier: on host panic, configured rproc's `ops->panic()` called before reboot.

## Architecture

`Rproc` lives in `drivers::remoteproc::Rproc`:

```
struct Rproc {
  dev: KernelDevice,
  name: KString,
  firmware: KString,
  ops: &'static dyn RprocOps,
  power: AtomicI32,           // boot refcount
  state: Atomic<RprocState>,
  lock: Mutex<()>,
  index: u32,
  domain: Option<IommuDomain>,
  has_iommu: bool,
  carveouts: ListHead<MemEntry>,
  mappings: ListHead<MemEntry>,
  rvdevs: ListHead<RprocVdev>,
  subdevs: ListHead<RprocSubdev>,
  traces: ListHead<RprocTrace>,
  cached_table: Option<KBox<ResourceTable>>,
  bootaddr: u64,
  recovery_disabled: AtomicBool,
  max_notifyid: i32,
  table_ptr: Option<NonNull<ResourceTable>>,
  features: u32,              // RPROC_FEAT_ATTACH_ON_RECOVERY, etc.
  cdev: KBox<Cdev>,
  coredump_conf: Atomic<CoredumpMode>,
}
```

Boot path `Rproc::boot`:
1. `atomic_inc_return(&rproc->power)` — if already booted, return.
2. `request_firmware(&fw, firmware, &rproc->dev)`.
3. `Elf::sanity_check(rproc, fw)` — validate ELF header bounds.
4. `Rproc::fw_boot(rproc, fw)`:
   a. `ops->parse_fw(rproc, fw)` — locate resource table.
   b. `Rproc::handle_resources(rproc, rproc_loading_handlers)` — for each rsc dispatch the per-type handler:
      - **CARVEOUT**: `rproc_handle_carveout` → defer alloc to `rproc_alloc_carveout`.
      - **DEVMEM**: per-devmem `iommu_map` into rproc->domain.
      - **TRACE**: register trace buffer in `rproc->traces` for debugfs.
      - **VDEV**: instantiate `rproc_vdev` + alloc vring carveouts (`rproc_alloc_vring`).
   c. `Rproc::alloc_registered_carveouts` — dma_alloc_coherent + optional IOMMU map.
   d. `Rproc::load_segments(fw)` — per-PT_LOAD `memcpy_toio` (or memcpy) into `da_to_va(da)`.
   e. `ops->start(rproc)` — release reset to co-processor.
5. State → RPROC_RUNNING.

ELF segment load `Elf::load_segments`:
1. Per PT_LOAD phdr (skip PT_NULL etc., skip memsz=0):
   - Validate `filesz ≤ memsz`.
   - Validate `offset + filesz ≤ fw->size`.
   - `da_to_va(rproc, da, memsz, &is_iomem)` — locate kernel VA inside a carveout.
   - `memcpy_toio(ptr, fw->data + offset, filesz)` if iomem else `memcpy`.
   - Zero-fill `[filesz, memsz)`.

Crash + recovery:
1. `rproc_report_crash(rproc, crash_type)` — log, transition to RPROC_CRASHED, queue `recovery_work` on `rproc_recovery_wq`.
2. Worker `rproc_crash_handler_work` → `rproc_boot_recovery(rproc)`:
   - `rproc_shutdown(rproc)` (skip free of stable carveouts if `RPROC_FEAT_ATTACH_ON_RECOVERY`).
   - `rproc_boot(rproc)` — reload firmware + restart.
3. On persistent failure: state → RPROC_OFFLINE, userland notified via uevent.

Sysfs surface (`remoteproc_sysfs.c`):
- `state` (RW): `start`, `stop`, `detach` commands.
- `firmware` (RW): firmware filename override.
- `name` (RO): rproc name.
- `recovery` (RW): `enabled`, `disabled`, `recovery-disabled`.
- `coredump` (RW): `default`, `inline`, `disabled`.

cdev surface (`remoteproc_cdev.c`):
- `/dev/remoteproc<N>`: ioctl `RPROC_SET_SHUTDOWN_ON_RELEASE` ties rproc lifetime to fd lifetime.

## Hardening

(Inherits from `drivers/remoteproc/00-overview.md` § Hardening.)

remoteproc-core specific reinforcement:

- **ELF header bounds strictly checked** — header size, phoff/shoff, phnum, per-phdr offset+filesz; refuse if any bound exceeds `fw->size`.
- **Per-segment `filesz ≤ memsz`** — defense against intentional smaller-on-disk attack.
- **Per-carveout `da_to_va` validates `[da, da+memsz)` ⊆ carveout** — refuses cross-carveout loads.
- **Per-rproc IOMMU domain isolates DMA** — `iommu_attach_device` mandatory if `has_iommu`; rejects boot if IOMMU alloc fails.
- **Resource-table size validated before dispatch** — per-rsc handler refuses entries smaller than declared rsc-type struct.
- **Per-vdev vring carveout DMA-coherent** — bounded by `RPROC_MAX_VRING` per vdev.
- **Recovery workqueue serializes per-rproc** — concurrent crash + boot races eliminated via `rproc_recovery_wq`.
- **Panic notifier order** — registered after critical-platform notifiers; bounded per-rproc invocation under panic-spin.
- **Coredump backing limited** — per-segment size bounded; defense against attack-fw triggering OOM via giant segments.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `rproc`, `rproc_mem_entry`, `rproc_vdev`, resource-table, vdev_buf, and coredump-staging slab caches; cdev/sysfs userland buffers bounded.
- **PAX_KERNEXEC** — `rproc_ops`, resource-handler dispatch tables, ELF-loader entry, IOMMU-fault callback, and recovery worker text live in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `rproc_boot`, `rproc_fw_boot`, ELF load, IOMMU fault, and recovery-worker entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `rproc->power` boot refcount, `rproc_vdev` refs, carveout-mem refs; overflow trap defeats boot/shutdown/recovery race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for carveout pages, vring backing memory, vdev_buf, resource-table cache, ELF-staging slabs; defense against firmware-leak across reload.
- **PAX_UDEREF** — SMAP/PAN enforced on every cdev/sysfs/debugfs entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `rproc_ops` (`prepare`, `start`, `stop`, `attach`, `kick`, `da_to_va`, `parse_fw`, `sanity_check`, `get_boot_addr`, `load`, `panic`), resource-handler table, and recovery-work fn marked `__ro_after_init` with kCFI typed dispatch.
- **GRKERNSEC_HIDESYM** — suppress `%p` of `rproc`, carveout VAs, IOMMU-domain pointers in tracepoints; gate kallsyms behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict ELF-sanity-fail, IOMMU-fault, crash, recovery-fail banners to CAP_SYSLOG so attackers cannot enumerate co-processor topology via dmesg.
- **Firmware signature mandatory** — `request_firmware_into_buf` chained with `kernel_read_file` signature check; refuse boot if signature missing/invalid when CONFIG_FW_LOADER_SYSFS_SIG=y.
- **ELF segment bounds vs. carveout** — every `da_to_va`-translated segment must lie strictly inside the declared carveout's `[da, da+len)`; refuse cross-carveout or out-of-table loads.
- **vdev_buf PAX_USERCOPY whitelist** — vring/vdev buffer caches whitelisted; reject userland copies that escape declared lengths.
- **Sysfs `state` / `firmware` / `recovery` writes require CAP_SYS_ADMIN** — refuse non-root toggles of boot state or firmware path.
- **Recovery rate-limit** — per-rproc recovery attempts within a sliding window bounded; persistent crash → state pinned RPROC_OFFLINE awaiting admin reset.
- **Coredump CAP gate** — coredump consumer (`/sys/class/devcoredump`) read requires CAP_SYS_ADMIN; coredump auto-expire after configurable timeout.

Rationale: a remoteproc is a second CPU sitting inside the same coherence domain as the host, often with peer-DMA to host memory. A loose ELF loader, an unbounded resource handler, a missed firmware signature, or a recovery-storm race lets an attacker replace co-processor firmware with arbitrary code that DMAs the host kernel. RAP/kCFI on `rproc_ops`, mandatory firmware signatures, strict ELF/carveout bounds, CAP_SYS_ADMIN on sysfs, recovery rate-limiting, and refcount-overflow trapping turn remoteproc from "trust the firmware" into a structural enforcement boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-vendor rproc drivers (qcom-q6v5, ti-k3, stm32, imx_rproc, etc. — separate Tier-3s)
- remoteproc-coredump details (covered in `remoteproc-coredump.md` future Tier-3)
- remoteproc-virtio (covered in `drivers/rpmsg/rpmsg-core.md` cross-ref)
- PRU-specific extensions (`pru_rproc.c` future Tier-3)
- Implementation code
