# Tier-3: drivers/gpu/drm/amd/amdgpu/{amdgpu_drv,amdgpu_device,amdgpu_kms,amdgpu_ioc32,amdgpu_pm,amdgpu_psp,amdgpu_atomfirmware,amdgpu_bios,amdgpu_discovery,amdgpu_vm,amdgpu_gtt_mgr,amdgpu_vram_mgr,amdgpu_object,amdgpu_bo_list,amdgpu_ttm,amdgpu_gem,amdgpu_cs,amdgpu_ctx,amdgpu_sched,amdgpu_ras}.c — AMD amdgpu core (lifecycle + DRM/KMS + memory + command-submission + RAS — every modern Radeon + Instinct + Ryzen iGPU)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpu/drm/00-overview.md
depends-on:
  - drivers/gpu/drm/drm-drv.md
  - drivers/gpu/drm/gem-core.md
  - drivers/gpu/drm/scheduler.md
upstream-paths:
  - drivers/gpu/drm/amd/amdgpu/amdgpu_drv.c                     (~3190 lines: drm_driver init, module params, PCI ID table, pm_ops)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_device.c                  (~6950 lines: per-device init/fini, IP block discovery+probe+init+fini, reset, recover, suspend/resume)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_kms.c                     (~2000 lines: KMS open/release, ioctl dispatch, info query, gem info)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_ioc32.c                   (32-bit ioctl compat)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_discovery.c               (~3000 lines: PSP IP-discovery binary parse → IP block enumeration)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_psp.c                     (~3500 lines: PSP firmware loader, secure-boot, TMR setup, TA load, TEE-equivalent for GPU)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_atomfirmware.c            (atomfirmware VBIOS table parse)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_bios.c                    (VBIOS retrieval — PCI ROM / ACPI / SoC)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c                      (~3000 lines: power-mgmt sysfs surface)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_vm.c                      (~3500 lines: per-VM page-table mgmt, PD/PT alloc, BO mapping)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_gtt_mgr.c                 (GTT (Graphics Translation Table — system-RAM-aperture-via-IOMMU) allocator)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_vram_mgr.c                (VRAM allocator — buddy-allocator over VRAM)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_object.c                  (~1500 lines: amdgpu_bo — TTM buffer object wrapper)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_bo_list.c                 (BO list for CS — per-CS reservation/eviction)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_ttm.c                     (~2300 lines: TTM resource manager hooks — VRAM/GTT/preempt-able/USER allocators)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_gem.c                     (~1100 lines: GEM ioctls — create, mmap, wait, va_map, va_unmap)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_cs.c                      (~2300 lines: command submission — userspace IB → fence → drm-sched job)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_ctx.c                     (~800 lines: per-userspace-process amdgpu_ctx — entity per ring)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_sched.c                   (drm-sched entity init)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_ras.c                     (~4500 lines: RAS — ECC error reporting, badpages list, recovery)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_xgmi.c                    (multi-GPU XGMI interconnect — for MI300 etc.)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_amdkfd.c                  (amdkfd compute-side interface)
  - drivers/gpu/drm/amd/amdgpu/amdgpu.h
  - include/uapi/drm/amdgpu_drm.h                                (UAPI ABI)
  - drivers/gpu/drm/amd/include/                                  (asic-shared headers)
  - drivers/gpu/drm/amd/display/                                  (DC — display engine, separate Tier-3)
  - drivers/gpu/drm/amd/pm/                                       (SMU — power-mgmt firmware interface, separate Tier-3)
  - drivers/gpu/drm/amd/amdkfd/                                   (compute interface, separate Tier-3)
  - drivers/gpu/drm/amd/amdgpu/gfx_v*, sdma_v*, vcn_v*, mes_v*, nbio_v*, gmc_v*, ih_v*, gfxhub_v*, mmhub_v*, ...   (per-ASIC IP block sources — ~150 files, ~80K lines)
-->

## Summary

amdgpu is the driver for every modern AMD GPU silicon: discrete Radeon RX (RDNA1/2/3/4 — RX 5000/6000/7000/9000 series), Radeon Pro (Vega Pro through Radeon Pro W7900), Instinct compute (MI100/MI200/MI300/MI325 datacenter accelerators), and every Ryzen iGPU (Vega-8/iGPU, RDNA1/2/3 inside Phoenix/Hawk Point/Strix Point/Krackan APUs). Tens of millions of deployed units globally. Used in gaming, professional workstation, ML/AI training (MI300X is a major competitor to NVIDIA H100), HPC, embedded.

The amdgpu source tree is the largest single driver tree in upstream Linux at ~320,000 lines across 304 .c files. The structure:

- **`amdgpu` core** (~30000 lines) — the parts this Tier-3 covers: PCIe lifecycle, IP block discovery+probe, KMS/DRM driver registration, memory mgmt (VM/VRAM/GTT/TTM/GEM/BO), command submission, ctx mgmt, RAS, PSP secure-boot.
- **`amdgpu` per-ASIC IP blocks** (~80000 lines, ~150 files) — `gfx_v9/10/11/12.c`, `sdma_v4/5/6/7.c`, `vcn_v1/2/3/4/5.c`, `mes_v10/11/12.c`, `nbio_v*`, `gmc_v*`, `ih_v*`, `gfxhub_v*`, `mmhub_v*`. Each IP block is the per-version implementation of one of the GPU's hardware engines (GFX = 3D engine, SDMA = system DMA, VCN = video codec, MES = micro-engine scheduler, NBIO = north-bridge IO, GMC = graphics memory controller, IH = interrupt handler, GFXHUB/MMHUB = page-table hubs). These IP blocks are out of scope for this Tier-3 — each gets its own iteration.
- **`amdgpu_dm`** (~80000 lines) — Display Core, the bridge between DRM and the DC (Display Core) firmware. Separate Tier-3.
- **`amd/pm/`** (~60000 lines) — SMU (System Management Unit) firmware interface. Separate Tier-3.
- **`amd/amdkfd/`** (~40000 lines) — HSA/ROCm compute interface. Separate Tier-3.
- **`amd/include/`** + auto-gen headers (~50000 lines) — ASIC register definitions, shared types.

This Tier-3 covers the ~30000-line core of amdgpu. Subsequent iterations will cover the IP block families (gfx, sdma, vcn, mes), display, SMU, amdkfd.

## Rust translation posture

amdgpu is the gnarliest driver Rookery will translate. The strategy must be very structured:

- **IP block abstraction.** Each IP block in upstream is a `struct amdgpu_ip_block_version` with `(funcs, type, major, minor, rev)`. Rookery: each IP block is a Rust crate (`amdgpu-gfx-v11`, `amdgpu-sdma-v6`, `amdgpu-vcn-v4`, etc.) implementing a common `IpBlock` trait. ASIC initialization composes the right set of crates per detected silicon.
- **Per-ASIC composition.** Upstream uses `discovery.c` to parse the PSP-loaded IP-discovery binary at probe; that tells the driver which IP blocks are present. Rookery dispatches the IP block crates by `(IpType, major, minor, rev)` lookup → composes the `AmdgpuDevice` at runtime.
- **AmdgpuDevice phase typing.** `AmdgpuDev<Discovered → IpBlocksProbed → IpBlocksInited → IpBlocksLateInited → Online>`.
- **TTM resource managers** as typed: `VramMgr`, `GttMgr`, `PreemptibleMgr`, `UserMgr` distinct types implementing a common `TtmResourceMgr` trait.
- **VM (per-userspace page-table) typed.** `Vm { root: Pde, ptes: PageTableTree, bos: BTreeMap<DevVa, Mapping> }`. PD/PT allocations are RAII guarded.
- **Command submission pipeline** typed: `CsIoctlRequest → ValidatedCs → BoList → Job → Fence`. Each step's output is the next step's input; cannot skip stages.
- **Fences as typed.** `Fence<S: FenceState>` with S in `{Unsignaled, Signaled, Error}`; signaling consumes the fence (closes UAF on fence reuse).
- **PSP firmware loader.** Typed pipeline; signature checks at typed boundaries.
- **RAS recovery state machine.** `Ras<Healthy → CorrectableErrSeen → Uncorrectable → RecoveryInProgress → RecoveryFailed>`.

The grsec/PaX section is mandatory. amdgpu has had multiple CVEs (CVE-2024-26975 RAS bad-page handling UAF, CVE-2023-2007 amdgpu_cs validation skip, multiple GEM ioctl bugs). GPU drivers are a high-CVE-density class because they expose huge ioctl surface to unprivileged userspace (every Wayland/Xorg session opens `/dev/dri/cardN` and issues thousands of ioctls per second).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct amdgpu_device` | top-level per-PCI-device state (~10K-line struct in upstream) | `AmdgpuDev<Phase>` |
| `struct amdgpu_ip_block` | per-IP-block instance (funcs + status) | `IpBlock` (trait) |
| `struct amdgpu_vm` | per-userspace page-table | `Vm` |
| `struct amdgpu_bo` | TTM-backed buffer object | `AmdgpuBo` |
| `struct amdgpu_ctx` | per-process ctx (entities per ring) | `Ctx` |
| `struct amdgpu_cs_parser` | per-CS-ioctl parser state | `CsParser` |
| `struct amdgpu_job` | drm-sched job (post-validation) | `Job` |
| `struct amdgpu_ring` | per-engine ring buffer (GFX/SDMA/COMPUTE/VCN/JPEG/etc.) | `Ring<EngineType>` |
| `struct amdgpu_ras` | per-device RAS state | `Ras<State>` |
| `amdgpu_pci_probe(pdev, ent)` / `_remove(pdev)` | PCI probe / remove | `AmdgpuDev::probe` / `Drop` |
| `amdgpu_device_init(adev, flags)` | per-device init | `AmdgpuDev::init` |
| `amdgpu_device_fini_*` | per-device fini | `AmdgpuDev::Drop` |
| `amdgpu_discovery_init(adev)` | parse IP-discovery binary | `Discovery::parse` |
| `amdgpu_discovery_set_ip_blocks(adev)` | compose IP block list | `Discovery::compose` |
| `amdgpu_device_ip_init(adev)` / `_late_init(adev)` / `_fini(adev)` | walk IP blocks calling their .init/.late_init/.fini | `IpBlockChain::init_all` etc. |
| `amdgpu_device_ip_set_clockgating_state` / `_powergating_state` | per-block power mgmt | `IpBlock::set_pm` |
| `amdgpu_device_gpu_recover(adev, job)` | GPU reset + recovery | `AmdgpuDev::recover` |
| `amdgpu_device_baco_enter` / `_exit` | BACO power-down/up | `Baco::enter` / `exit` |
| `amdgpu_psp_init(adev)` / `_load_fw(adev)` / `_load_ta(adev)` | PSP boot, FW load, TA load | `Psp::init` / etc. |
| `amdgpu_atomfirmware_*` | VBIOS table parse | `Atomfw::*` |
| `amdgpu_vm_init(adev, vm)` / `_fini(adev, vm)` / `_alloc_pts(...)` / `_bo_map(...)` / `_bo_unmap(...)` | per-VM | `Vm::init` / etc. |
| `amdgpu_bo_create(adev, bp, bo_ptr)` / `_bo_destroy(...)` / `_bo_pin(...)` / `_bo_kmap(...)` | BO lifecycle | `AmdgpuBo::create` / `Drop` |
| `amdgpu_ttm_init(adev)` / `_tt_create(...)` / `_tt_destroy(...)` / `_bo_evict(...)` / `_bo_move(...)` | TTM hooks | `Ttm::*` |
| `amdgpu_gem_create_ioctl` / `_gem_mmap_ioctl` / `_gem_wait_idle_ioctl` / `_gem_va_ioctl` / `_gem_op_ioctl` / `_gem_metadata_ioctl` / `_gem_userptr_ioctl` | GEM ioctls | `Gem::*` |
| `amdgpu_cs_ioctl` | command submission | `Cs::ioctl` |
| `amdgpu_cs_parser_init(...)` / `_parse(...)` / `_validate(...)` / `_submit(...)` | CS pipeline | `CsParser::*` |
| `amdgpu_ctx_init(adev, priority, file_priv, ctx)` / `_ioctl(...)` / `_fini(...)` | ctx mgmt | `Ctx::*` |
| `amdgpu_ctx_get_entity(...)` | entity-per-ring lookup | `Ctx::get_entity` |
| `amdgpu_ras_init(adev)` / `_recovery_init(...)` / `_query_error_status(...)` / `_eeprom_read(...)` | RAS | `Ras::*` |
| `amdgpu_ras_recovery_work(work)` | RAS recovery handler | `Ras::recovery_work` |
| `amdgpu_xgmi_add_device(adev)` / `_remove_device(adev)` / `_set_pstate(adev, pstate)` | XGMI multi-GPU | `Xgmi::*` |
| `amdgpu_amdkfd_device_init(adev)` / `_device_fini(adev)` | amdkfd hand-off | `Amdkfd::init_hook` |
| `amdgpu_amdkfd_pre_reset(adev)` / `_post_reset(adev)` | reset coordination | `Amdkfd::reset_hooks` |

## Compatibility contract

REQ-1: PCI ID table — every AMD GPU silicon from Bonaire (CIK) through Phoenix4/Strix Halo + MI325. Frozen against `pciidlist[]` in `amdgpu_drv.c`.

REQ-2: drm_driver registration with DRIVER_GEM | DRIVER_RENDER | DRIVER_MODESET | DRIVER_SYNCOBJ caps. UAPI version stays at the version reported in `KMS_DRIVER_*` constants.

REQ-3: IP discovery — PSP loads the discovery binary at boot; driver parses it to enumerate IP blocks present on this specific silicon. Per-silicon IP block list determined dynamically — there is no per-asic-id table in modern AMD drivers.

REQ-4: KMS/atomic — full atomic-modesetting compliance; cursor planes, primary plane, overlay planes per spec. (Detailed display logic in amdgpu_dm — separate Tier-3.)

REQ-5: BO mgmt — TTM-backed amdgpu_bo objects in domains: VRAM (device-local), GTT (system RAM via IOMMU aperture), CPU (system RAM, host-visible). Eviction policy per TTM.

REQ-6: VM (per-process page-table) — multi-level page tables in VRAM (preferred) or GTT (fallback). PDE/PTE format per ASIC generation. BOs mapped at userspace-chosen virtual addresses via `AMDGPU_GEM_VA` ioctl.

REQ-7: Userptr — `AMDGPU_GEM_USERPTR` maps host memory into the GPU VM through HMM (mmu-notifier-backed); supports unified-virtual-memory programming models.

REQ-8: Command submission (`AMDGPU_CS`) — userspace assembles an Indirect Buffer (IB) of GPU commands + a list of BOs touched + dependency syncobjs/fences; ioctl validates, allocates a job, submits to drm-sched; returns a fence handle.

REQ-9: Context (`AMDGPU_CTX`) — per-process state: priority, per-ring entities, GPU reset cookie (so userspace knows when the GPU died and reset). Multiple ctxs per process supported.

REQ-10: SyncObj — DRM syncobj integration for cross-process fence sharing (used by Vulkan timeline semaphores, Wayland present-feedback).

REQ-11: PSP secure-boot — at probe, driver loads PSP FW which loads SOS (Secure OS) which loads sub-FWs (SMU, ASD, TA — Trusted Apps for HDCP, RAS, etc.). Signature verification at each step.

REQ-12: RAS — ECC error reporting via sysfs `/sys/class/drm/cardN/device/ras/`; correctable errors logged, uncorrectable errors trigger recovery. Bad-page list maintained in eeprom on MI200+.

REQ-13: XGMI — multi-GPU coherent interconnect for MI200/300 multi-package setups; XGMI link bring-up + per-node connectivity discovery.

REQ-14: amdkfd hand-off — compute-mode KFD ioctls (`/dev/kfd`) sit on amdgpu; KFD calls back into amdgpu for queues + memory + GPU reset.

REQ-15: GPU recovery — on hang detection (timeout-detection-and-recovery TDR per-ring), `amdgpu_device_gpu_recover` triggers: stop schedulers → wait for in-flight → soft/hard reset → re-init IP blocks → re-arm rings → resume schedulers. Recovery coordinated with amdkfd.

REQ-16: SR-IOV — datacenter SKUs (MI series) support SR-IOV; per-VF amdgpu instance. Hostside (PF) coordinates GPU partitioning via PSP TA.

REQ-17: Suspend/resume + runtime PM — full S3/S4 + runtime PM with BACO (Bus Active power-Off) for hybrid graphics laptops.

## Acceptance Criteria

- [ ] AC-1: `lspci -d 1002:` shows AMD GPU; Rookery probes; PSP boot completes; IP discovery enumerates expected IP blocks for the silicon; `dmesg | grep amdgpu` matches upstream init.
- [ ] AC-2: Display: KMS-only smoke (any framebuffer renders to monitor through DC).
- [ ] AC-3: 3D rendering: Mesa Radv/Radeon driver runs `vkcube`; ~60 fps on a discrete GPU.
- [ ] AC-4: Compute via HIP/ROCm: HIP sample (`vector_add`) runs end-to-end on MI200; output validated.
- [ ] AC-5: AMDKFD: ROCm-smi reports the GPU; KFD-spec workloads run.
- [ ] AC-6: GPU recovery: trigger a GFX hang via Mesa-test-driven shader-loop; TDR fires; reset completes; subsequent workloads run.
- [ ] AC-7: Userptr: HSA application with `MAP_USER_PTR` maps host memory; GPU reads/writes succeed; CPU sees the updates.
- [ ] AC-8: RAS: on MI200, ECC scrubbing on; correctable errors logged; bad-page list persisted to eeprom.
- [ ] AC-9: XGMI: 4-GPU MI300 chassis; XGMI links up; cross-GPU memory access via amdkfd.
- [ ] AC-10: SR-IOV: MI200 PF carved into 4 VFs; each VF presented to a KVM guest; per-VF rendering works.
- [ ] AC-11: Suspend/resume: S3 cycle preserves all open ctxs + GEM objects; first frame after resume renders cleanly.

## Architecture

**IP block composition.** Modern AMD GPUs are SoCs with up to 20+ hardware IP blocks (GFX, SDMA, VCN-decode, VCN-encode, JPEG, MES, GMC, MMHUB, GFXHUB, IH, NBIO, PSP, SMU, DCE/DCN, etc.). Each IP block has multiple per-silicon versions. The driver does not hard-code "for MI300 use gfx_v9_4_3 + sdma_v4_4_2 + ..." — instead, the PSP at boot exposes an "IP discovery binary" listing exactly which IP blocks at which versions are present on this silicon. `amdgpu_discovery.c` parses it; `amdgpu_discovery_set_ip_blocks` walks the parsed list and registers each.

Rookery: each IP block is its own crate (`amdgpu-gfx-v11`, `amdgpu-sdma-v6`, etc.) implementing the `IpBlock` trait:

```rust
trait IpBlock {
    fn ip_type(&self) -> IpType;
    fn version(&self) -> (u8, u8, u8);
    fn early_init(&mut self, dev: &AmdgpuDev) -> Result<(), IpError>;
    fn sw_init(&mut self, ...) -> Result<(), IpError>;
    fn hw_init(&mut self, ...) -> Result<(), IpError>;
    fn late_init(&mut self, ...) -> Result<(), IpError>;
    fn suspend(&mut self, ...) -> Result<(), IpError>;
    fn resume(&mut self, ...) -> Result<(), IpError>;
    fn hw_fini(&mut self, ...) -> Result<(), IpError>;
    fn sw_fini(&mut self, ...) -> Result<(), IpError>;
    fn set_powergating(&mut self, on: bool) -> Result<(), IpError>;
    fn set_clockgating(&mut self, on: bool) -> Result<(), IpError>;
}
```

Per-ASIC composition at runtime: parse discovery → instantiate the right `Box<dyn IpBlock>` instances → chain them.

**AmdgpuDev phase typing.** `Discovered → IpBlocksProbed → IpBlocksInited → IpBlocksLateInited → Online`. The chain of init phases mirrors upstream's `ip_init` / `ip_late_init` ordering. Phases that need a specific subset of IP blocks (e.g. CS submission needs GFX inited) are typed to require the right markers.

**VM (per-process page table).** Each open `/dev/dri/cardN` fd gets its own VM. The VM owns a multi-level page-table tree (typically 4 levels on modern ASICs, 5 levels for very large address spaces). PD/PT entries point to lower-level tables or to ASIC-physical addresses (VRAM, GTT, MMIO aperture). BOs map at userspace-chosen virtual addresses via the `AMDGPU_GEM_VA` ioctl. Update batching: `VM_UPDATE` schedules a PT-update job through drm-sched so multiple maps in flight don't fight for the PT-update SDMA engine.

**TTM resource managers.** `VramMgr` (buddy-allocator over VRAM, possibly with contiguous-CMA support for video-codec buffers), `GttMgr` (LRU-managed system-RAM aperture, mapped through the GPU's IOMMU), `PreemptibleMgr` (system RAM that can be paged back out for KFD compute mode), `UserMgr` (HMM-pinned host pages for userptr BOs). All implement TTM's resource-mgr API.

**Command submission pipeline.** `amdgpu_cs_ioctl` workflow:
1. Parse — split the input chunks (IB chunks, BO_LIST chunks, dependency chunks, syncobj chunks). Validate sizes and indices.
2. Build BO list — for each BO referenced, take a reservation. Detect over-commit (more BOs than fit in VRAM); request eviction.
3. Validate — for each IB chunk, check that the IB BO is mapped, the IB size fits, the IP block (which engine) is valid.
4. Build job — allocate an `amdgpu_job`, fill in the IB pointers, hook up the dependency fences.
5. Submit — push the job onto the drm-scheduler entity for the target ring. Returns immediately with a fence handle.
6. drm-scheduler picks up the job when the engine is idle and the dependencies are signaled, then executes (rings the doorbell of the target engine).

Rookery models this as `CsParser → ValidatedCs → BoList → Job → Fence` — each step's output is the next's input, no skipping.

**Fences.** GPU fence types: HW fence (signaled by GPU writing a 64-bit value), userspace-fence wait (cross-process via syncobj). Rookery's typed `Fence<S>` makes UAF impossible (signaled fence can't be re-signaled; an unsignaled fence can't be dropped without explicit cancel).

**PSP secure-boot.** PSP (Platform Security Processor) is an on-die ARM core that runs AMD's Secure OS. Driver flow:
1. Driver loads PSP FW from `/lib/firmware/amdgpu/<asic>_sos.bin` etc.
2. PSP boots SOS.
3. SOS loads sub-FWs: SMU, ASD (AMD Secure Driver), TAs (Trusted Apps: HDCP, RAS, XGMI, RAP).
4. PSP exposes a mailbox interface; driver sends commands via the mailbox to TAs.
5. Signature verification at every step (RSA per AMD signing key).

Rookery models the PSP interaction as a typed pipeline: `Psp<Pre> → Psp<SosLoaded> → Psp<TaLoaded>` with per-stage validation.

**RAS recovery.** ECC errors detected by various IP blocks raise interrupts that funnel through IH (interrupt handler). RAS subsystem categorizes: correctable (log, increment counter) or uncorrectable (trigger recovery). Uncorrectable errors: identify the failing BO/process, kill the offending ctx, possibly trigger GPU reset. Bad-page list: pages with permanent UE are recorded in on-board eeprom and refused-for-alloc going forward.

**GPU recovery on hang.** Per-ring TDR (timeout-detection-and-recovery): if a job has been running >tdr_timeout, the scheduler queues the ring for reset. Reset variants: soft reset (engine-level), mode-1 reset (per-IP block), mode-2 reset (whole-device, no PSP), baco reset (power-off-power-on without losing PCI link). Recovery walks: pause schedulers → wait quiesce → identify scope → reset → re-init IP blocks affected → re-arm rings → resume. amdkfd is notified pre/post so KFD queues can be re-armed.

## Hardening

- IP discovery binary parser bounded (size cap, signature-validated by PSP).
- VM page-table walks bounded by tree depth.
- BO size and alignment validated at create.
- CS ioctl IB chunk count + size validated; per-chunk size cap.
- BO list reservation atomic (all-or-nothing); avoid partial-mapping states.
- All TTM resource-mgr allocations bounded by per-domain caps.
- PSP cmd timeouts (10s typical); past timeout, recovery fires.
- Fence wait paths use `dma_fence_wait_timeout` with mandatory timeout.
- Ras recovery rate-limited; >3 in 60s → permanent down for the affected domain.
- XGMI link state machine has explicit timeout per state.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — every ioctl in amdgpu has a typed argument struct; copy_from_user uses sizeof() of the typed struct (with per-version size compat for backward compat). CS ioctl chunk parsing uses bounded reads.
- **PAX_KERNEXEC** — `amdgpu_drm_driver` ops, ioctl dispatch table, IP block .funcs pointers, PSP cmd dispatch, RAS callback tables all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — every ioctl entry path (GEM, CS, CTX, INFO, PM, RAS) per-syscall stack-rerandomized.
- **PAX_REFCOUNT** — `amdgpu_device` refcount, per-`amdgpu_bo` refcount, per-`amdgpu_ctx` refcount, per-VM refcount, per-fence refcount, per-IP-block refcount use saturating refcounts; underflow at remove is a hard panic.
- **PAX_MEMORY_SANITIZE** — BO backing pages cleared on TTM-evict (may contain pixel data, neural network weights, financial app screenshots). PSP cmd buffers cleared. PT-update buffers cleared.
- **PAX_UDEREF** — SMAP/SMEP on every ioctl + sysfs + debugfs entry.
- **PAX_RAP / kCFI** — `drm_driver` ops, ioctl table, IP block funcs (PFN tables that vary per-ASIC), PSP cmd dispatch, RAS callbacks all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/dri/N/amdgpu_*`) scrubbed for non-root.
- **GRKERNSEC_DMESG** — verbose amdgpu logs (CS parsing trace, RAS error details with addresses, GPU reset traces) restricted from non-root.
- **DRM_AUTH gating for privileged ioctls** — `DRM_MASTER`-required ioctls (mode set, etc.) gated. Render-only ioctls (CS, GEM) accessible to authenticated render clients.
- **IOMMU mandatory for GTT / userptr** — GTT and userptr both DMA from system pages. Without an IOMMU enforcing per-device permissions, a malicious userspace process could craft an IB that DMA-writes into kernel memory via the GPU. Refuse probe if IOMMU isolation not reported.
- **IB chunk size cap strict** — IB chunks capped at sane size (4 MB upstream); closes a path where unprivileged userspace pushes giant IBs to exhaust kernel memory.
- **CS validation cannot be bypassed** — CS ioctl validation is a closed pipeline; the `ValidatedCs` type can only be constructed from `CsParser::validate`. Closes CVE-2023-2007 class (skipping validation on CS error paths).
- **PSP FW signature mandatory** — every FW image (SOS, SMU, ASD, TA, GFX-MEC, SDMA-FW, etc.) signature-checked by PSP before activation. Unsigned FW refused even with CAP_SYS_RAWIO. LOCKDOWN_INTEGRITY_MAX disables FW updates entirely.
- **Userptr HMM mmu-notifier strict** — `amdgpu_gem_userptr` registers an mmu-notifier; on host VMA change, GPU mappings invalidated synchronously. A race where the host munmap()s before the GPU sees the invalidation is closed by holding the BO reservation across the invalidate.
- **VM PT allocations capped** — per-VM total PT memory capped (driver default + sysfs override). Closes "userspace creates 10^9 BOs to OOM the kernel through PT growth" class.
- **Bad-page list signed** — on MI200+, the bad-page list stored in on-board eeprom is signed by PSP; a tampered list is refused. Closes a path where a compromised admin could mark good pages bad to deny-of-service.
- **RAS recovery scoped** — RAS recovery only resets the IP blocks affected by the error, never the whole device unless necessary. Closes "single ECC error tears down all running workloads" amplification.
- **GPU recovery rate-limit** — >3 full-device resets in 60s → permanent failure declared. Closes "compromised TA gaslights us into endless reset" class.
- **XGMI peer-link allowlist** — only signed XGMI partner devices accepted. Closes a "rogue PCIe device claims to be XGMI peer" attack class.
- **amdkfd SVM (Shared Virtual Memory) gated** — SVM gives the GPU access to the entire process virtual address space; requires explicit per-process opt-in + CAP_SYS_NICE.
- **DRI fd file_operations kCFI** — `amdgpu_kms_fops`'s open/release/ioctl/poll/mmap pointers all kCFI-signed.
- **mmap of GEM objects W^X enforced** — BO mmaps default NX; only explicit AMDGPU_GEM_DOMAIN_GDS|GWS|OA require special handling (those domains are GPU-only).
- **Per-VM tenant isolation in SR-IOV** — VFs cannot read other VFs' VRAM, VM mappings, or GFX contexts. Enforced by PSP TA-managed VRAM partitioning + GPU's VMID isolation.
- **Display Core (DC) FW signature mandatory** — same as other FW.

Per-doc rationale: amdgpu has 320K LOC and 304 .c files — the largest single driver in upstream Linux. The huge ioctl surface (GEM + CS + CTX + INFO + AMDKFD) + the privileged-FW-update path + multi-GPU XGMI + SR-IOV + the always-on PSP secure boot create a massive attack surface. Historical CVEs (CVE-2024-26975 RAS UAF, CVE-2023-2007 CS validation skip) show the bug classes. The Rust translation's structural defenses — IP block crates as separate compilation units, typed CS pipeline that cannot skip validation, typed Fence<S> that closes UAF, typed Vm with capped PT memory, typed Psp pipeline with per-stage signature checks — close the dominant historical bug classes. The IOMMU-mandatory gate is structural (GTT + userptr both DMA from system memory; without per-device IOMMU isolation, the GPU is a DMA-attack vector against arbitrary kernel memory). PSP FW signature mandatory is also structural — the whole secure-boot chain depends on signature checks; removing them is equivalent to giving any local root unconstrained RCE on the PSP, which has full access to GPU memory and command rings.

## Open Questions

- [ ] Q1: How to scope subsequent iterations — should each IP block family (gfx, sdma, vcn, mes, gmc, ih, nbio, etc.) be its own Tier-3, or should they be grouped by ASIC generation (GFX9 doc covers gfx_v9, sdma_v4, vcn_v1, ...)? Recommendation: by IP family + per-version; sdma_v6 is its own doc covering sdma_v6_0 + sdma_v6_1.
- [ ] Q2: amdgpu_dm (Display Core) is ~80K lines and tightly couples to AMD's closed-source DC HAL (Display Core HAL is GPL but autogenerated from internal AMD specs). Rookery's display strategy: reimplement DC from scratch (multi-year effort) or interface to the existing DC HAL? Recommendation: interface initially, plan reimplementation as a separate track.
- [ ] Q3: SMU (System Management Unit) firmware interface is heavily ASIC-specific. Should the SMU doc be one per-version (smu_v11, smu_v13, smu_v14) or one big SMU doc? Recommendation: one per-version, smu_v13 is the highest priority (RDNA3 + MI300).
- [ ] Q4: SR-IOV PSP-mediated GPU partitioning — well-defined ABI between host PF and PSP TA. Should the Rust translation reimplement the host-side or interface to the existing AMD-provided code? Strong recommendation: reimplement (the host-PSP-TA interface is small + critical for tenant isolation).
- [ ] Q5: amdkfd vs amdgpu integration — historically separate kmods, increasingly integrated. Rookery should have them as one binary but separate crates with a clean trait interface.
- [ ] Q6: GPU recovery testing — needs full per-IP-block reset coverage. Should the test matrix include intentional shader hangs across every IP block we support?

## Verification

- **Kani SAFETY**: prove the CS validation pipeline cannot be skipped (typed ValidatedCs only constructible from validate()). Prove Fence<Unsignaled>::signal consumes self into Fence<Signaled> (no double-signal). Prove per-VM PT memory cap cannot be exceeded.
- **TLA+**: model the GPU recovery state machine with concurrent CS submission + reset + amdkfd reset hook. Check no in-flight CS can complete during reset (would corrupt). Check all in-flight CSs are returned as -ECANCELED after reset.
- **Verus**: functional spec of CS validation — for any input CS_ioctl args, either rejects with -EINVAL or produces a ValidatedCs whose IBs all reference mapped BOs at valid VAs.
- **Kani+Verus**: invariant that every BO created has a destroyer (no leaks at remove). Invariant that every fence created is either signaled, errored, or its waiter is woken at destroy.
- **Integration**: Mesa Radv vkCTS run (Vulkan compliance), HIP/ROCm sample test matrix, glmark2 (3D), `vainfo` + ffmpeg vaapi decode (VCN), GPU recovery injection at every IP block, SR-IOV passthrough.
- **Fuzz**: CS ioctl fuzzer (mutate chunks, IB pointers, BO list contents) — should never produce a kernel oops. GEM ioctl fuzzer. RAS event payload fuzzer.
- **Penetration**: unprivileged process attempts CS with crafted IB targeting other process's VM — refused by VM isolation. Unprivileged attempts to load unsigned FW — refused.

## Out of Scope

- Per-ASIC IP block sources (gfx_v9..v12, sdma_v4..v7, vcn_v1..v5, mes_v10..v12, gmc_v9..v12, nbio, ih, etc.) — separate Tier-3 docs per IP family (planned next iterations)
- amdgpu_dm (Display Core) — separate Tier-3
- amd/pm/ (SMU) — separate Tier-3 per SMU version
- amd/amdkfd/ (compute) — separate Tier-3
- DRM mid-layer (`drivers/gpu/drm/*`) — covered in `drivers/gpu/drm/00-overview.md` + drm-drv/gem-core/scheduler
- TTM mid-layer (`drivers/gpu/drm/ttm/*`) — separate Tier-3
- Older AMD radeon driver (`drivers/gpu/drm/radeon/`) — legacy, separate Tier-3 (lower priority)
- DC HAL internal details — AMD-internal, out of scope (see Q2)
