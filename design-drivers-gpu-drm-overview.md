---
title: "Tier-2: drivers/gpu/drm — Direct Rendering Manager (core + GEM + KMS + atomic + scheduler + TTM + per-vendor amdgpu/i915/xe/nouveau/radeon/v3d/vc4/msm/panfrost/panthor/lima/qxl/virtio-gpu/vmwgfx)"
tags: ["tier-2", "drivers", "drm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the DRM subsystem — Linux's GPU + display abstraction layer underneath Mesa, the Wayland/X11 servers, KMSCon, libdrm, libgbm, and every userspace graphics + compute consumer (OpenGL, Vulkan, OpenCL, OpenCV-on-GPU, ROCm, CUDA-via-zluda, llama.cpp Vulkan, Blender Cycles, ffmpeg HW codec). Three concentric scopes:

- **DRM core** (`drm_*.c` at top level, ~110 files): the framework — `struct drm_device` + `drm_driver_register`, `drm_open`/`drm_release` chardev `/dev/dri/card<N>` + `/dev/dri/renderD<N>`, ioctl dispatch, `struct drm_file` per-fd state, drm_mm range-allocator, drm_buddy memory allocator, drm_exec lock-acquire helper, drm_syncobj fence-handle, drm_lease (split-screen lease to client), drm_property abstraction, drm_writeback (CRTC writeback to memory), atomic-modeset core, atomic-helpers, EDID parser, DisplayID parser, DPMS, color-management, bridge / panel / encoder / connector framework, MIPI-DSI / HDMI / DisplayPort / VGA / LVDS connectors, fbdev compatibility shim
- **Memory managers**: GEM (Graphics Execution Manager — generic buffer-object framework underneath every modern driver), TTM (Translation Table Maps — shmem-+ video-RAM-aware GEM backend used by amdgpu/i915/xe/nouveau/radeon/qxl/virtio-gpu/vmwgfx), Shmem-helper (CMA-/system-memory-only GEM for SoC GPUs), DMA-helper (physically-contiguous GEM for ARM SoC display)
- **Scheduler**: `drm_sched` — generic GPU command-submission scheduler with priority + dependency/fence chaining; consumed by amdgpu, etnaviv, lima, msm, panfrost, panthor, v3d
- **KMS / display**: atomic-modeset state machine, plane / CRTC / connector / encoder / bridge / panel object hierarchy, mode validation, hotplug detection, color-space + HDR metadata, VRR (Variable Refresh Rate)
- **Display sub-tree** (`display/`): DP MST hub support, HDCP, DP tunnel
- **Bridge sub-tree** (`bridge/`): per-bridge IC drivers (dw-hdmi, ti-sn65dsi83, lt9611, anx7xx, etc.) — mostly ARM SoC
- **Panel sub-tree** (`panel/`): per-panel drivers (mostly ARM tablets/laptops)
- **Per-vendor render+display drivers**:
  - **amdgpu** (`amd/amdgpu/` + `amd/display/` + `amd/include/` + `amd/pm/` + `amd/ras/` + `amd/amdkfd/` + `amd/amdxcp/` + `amd/acp/`): the AMD universal driver covering all GCN1.0+ through RDNA4+ GPUs (Vega, Navi, RDNA, RDNA2, RDNA3, RDNA4, MI100/MI200/MI300, Phoenix/Strix iGPU, Krackan APUs). Includes `amdgpu` (render+display+compute), `amdkfd` (Heterogeneous Compute compute-only chardev for ROCm — `/dev/kfd`), `amdxcp` (XCP — Cross-CCD-Partitioning for MI300), `acp` (Audio CoProcessor for AMD APUs)
  - **i915** (`i915/`): Intel HD/Iris/Arc graphics — Gen2 through Gen12.5 (Iris Xe / Alchemist / Battlemage). Largest single GPU driver in tree
  - **xe** (`xe/`): Intel new-gen-only driver (Xe2+ / Lunar Lake / Battlemage discrete) — replaces i915 for newer hardware
  - **nouveau** (`nouveau/`): community NVIDIA driver (legacy NV04 through Turing/Ampere/Ada user-space-driver via NVK)
  - **radeon** (`radeon/`): legacy AMD pre-GCN driver (R100-Northern-Islands)
  - **virtio** (`virtio/`): virtio-gpu paravirt GPU (qemu / cloud-hypervisor 2D + 3D modes; cross-domain-context for guest 3D pass-through)
  - **vmwgfx** (`vmwgfx/`): VMware SVGA virtual GPU
  - **qxl** (`qxl/`): QEMU Spice virtual GPU
  - **v3d** (`v3d/`) + **vc4** (`vc4/`): Broadcom VideoCore (Raspberry Pi)
  - **msm** (`msm/`): Qualcomm Adreno (Snapdragon)
  - **panfrost** (`panfrost/`) + **panthor** (`panthor/`): Arm Mali (panfrost = Bifrost/Midgard, panthor = newer CSF-firmware Mali)
  - **lima** (`lima/`): Arm Mali Utgard (older mobile)
  - **etnaviv** (`etnaviv/`): Vivante GC series
  - **rockchip** (`rockchip/`), **rcar-du** (`rcar-du/`), **mediatek** (`mediatek/`), **ingenic** (`ingenic/`), **stm** (`stm/`), **sun4i** (`sun4i/`), **tegra** (`tegra/`), **exynos** (`exynos/`), **kirin** (`hisilicon/`), **omap** (`omapdrm/`), **xlnx** (`xlnx/`), **logicvc**, **mxsfb** — ARM SoC display (compile-gated off for v0)
  - **ast** (`ast/`): ASPEED AST BMC display
  - **mgag200** (`mgag200/`): Matrox MGA G200 (server BMC)
  - **bochs** (`tiny/bochs.c`): Bochs / qemu VGA std-vga
  - **cirrus** (`tiny/cirrus.c`): Cirrus Logic GD5446 (qemu legacy)
  - **gud** (`gud/`): Generic USB Display
  - **adp** (`adp/`): Apple AGX Display Pipeline (M-series — keep watching upstream)
- **Helper sub-trees**: `display/` (DP MST + HDCP + DP-tunnel + DSC + AUX channel), `bridge/`, `panel/`, `tests/` (KUnit), `tiny/` (small fb-style legacy drivers), `clients/` (in-kernel splash + early console), `ci/` (CI configs)
- **fbdev compat shim**: `drm_fb_helper.c` + `drm_fbdev_*` glues legacy framebuffer-console to atomic KMS so /dev/fb0 + fbcon still work while DRM owns the hardware

Heavily cross-referenced from `kernel/dma/00-overview.md` (dma-buf cross-driver buffer share + dma-fence + dma-resv), `drivers/pci/00-overview.md` (PCIe binding + GPU-direct via P2PDMA + SR-IOV PF/VF), `drivers/iommu/00-overview.md` (per-process GPU-VM bindings via PASID + SVA), `drivers/vfio/00-overview.md` (vfio-mdev mediated GPU + vfio-pci passthrough), `drivers/usb/00-overview.md` (USB displays via gud + USB DisplayLink), `sound/00-overview.md` (HDA-codec on display port), `kernel/sched/00-overview.md` (drm-cgroup priority + GPU-deadline), `kernel/cgroup/00-overview.md` (drm cgroup controller for memory + scheduling), `mm/00-overview.md` (GEM/TTM page allocations + PSI integration).

### Out of Scope

- Per-Tier-3 (Phase D) — per-driver docs arrive incrementally
- ARM SoC display / GPU drivers beyond panfrost/panthor/lima/v3d/vc4 (`armada/`, `aspeed/` keep, `atmel-hlcdc/`, `etnaviv/`, `exynos/`, `imagination/`, `imx/`, `ingenic/`, `mediatek/`, `meson/`, `msm/` keep, `omapdrm/`, `pl111/`, `rcar-du/`, `renesas/`, `rockchip/`, `solomon/`, `sprd/`, `stm/`, `sti/`, `sun4i/`, `tegra/`, `tidss/`, `tilcdc/`, `tve200/`, `xlnx/`, `loongson/`, `logicvc/`, `mxsfb/`, `vboxvideo/`) — most compile-gated off
- 32-bit-only paths
- vbox-video (out of v0)
- Implementation code

### components

### Core (top-level `drm_*.c`)

- **drm_drv** (`drm_drv.c`): `struct drm_driver` registration, drm_dev_register/unregister, drm_dev_alloc, drm-rules-of-engagement
- **drm_file / drm_ioctl** (`drm_file.c` + `drm_ioctl.c`): per-fd state, ioctl dispatch table
- **drm_atomic** (`drm_atomic.c` + `drm_atomic_helper.c` + `drm_atomic_state_helper.c` + `drm_atomic_uapi.c`): atomic-modeset state machine + helpers + UAPI commit
- **drm_crtc / drm_plane / drm_connector / drm_encoder / drm_bridge / drm_panel** (`drm_crtc.c`, `drm_plane.c`, `drm_connector.c`, `drm_encoder.c`, `drm_bridge.c`, `drm_panel.c` — but bridges/panels mostly in `bridge/` + `panel/` subdirs): KMS object hierarchy
- **drm_mode_object / drm_property / drm_blend / drm_color_mgmt / drm_colorop** (`drm_mode_object.c`, `drm_property.c`, `drm_blend.c`, `drm_color_mgmt.c`, `drm_colorop.c`): KMS property + blend + color pipeline
- **drm_edid / drm_displayid / drm_eld** (`drm_edid.c` + `drm_edid_load.c` + `drm_displayid.c` + `drm_eld.c`): EDID + DisplayID parsing + EDID firmware load + ELD construction
- **drm_modes / drm_modeset_lock / drm_modes_helper** (`drm_modes.c`, `drm_modeset_lock.c`, `drm_modeset_helper.c`): mode descriptor + ww-mutex modeset locks
- **drm_gem** (`drm_gem.c` + `drm_gem_dma_helper.c` + `drm_gem_shmem_helper.c` + `drm_gem_atomic_helper.c` + `drm_gem_framebuffer_helper.c` + `drm_gem_vram_helper.c` + `drm_gem_ttm_helper.c`): GEM core + per-backend helpers
- **drm_buddy / drm_mm / drm_exec / drm_suballoc** (`drm_buddy.c`, `drm_mm.c`, `drm_exec.c`, `drm_suballoc.c`): allocator + range-tracker + lock-acquire helper + sub-allocator
- **drm_syncobj / drm_fence / drm_writeback** (`drm_syncobj.c`, `drm_writeback.c`): fence-handle UAPI + writeback CRTC
- **drm_dp_helper / drm_dp_mst_topology / drm_dp_aux_bus / drm_dp_dual_mode_helper / drm_dp_helper_internal / drm_dp_helper / drm_dp_mst** (`display/drm_dp_*` etc.): DisplayPort helper + MST topology mgmt
- **drm_hdcp_helper** (`display/drm_hdcp_helper.c`): HDCP 1.4 + 2.2/2.3
- **drm_format / drm_fourcc / drm_format_helper** (`drm_format_helper.c`, `drm_fourcc.c`): pixel-format introspection
- **drm_framebuffer** (`drm_framebuffer.c`): framebuffer object lifecycle
- **drm_lease** (`drm_lease.c`): client lease (compositor lets sub-client own a CRTC)
- **drm_vblank** (`drm_vblank.c` + `drm_vblank_work.c`): vblank events + per-vblank work
- **drm_vma_manager** (`drm_vma_manager.c`): per-fd mmap-offset allocator
- **drm_print / drm_debugfs / drm_debugfs_crc / drm_dumb_buffers / drm_draw / drm_flip_work / drm_format_helper / drm_writeback / drm_self_refresh_helper / drm_simple_kms_helper / drm_drv_internal / drm_internal**: misc helpers
- **drm_atomic_state_helper / drm_atomic_helper**: atomic-modeset reusable callbacks
- **drm_fbdev_dma / drm_fbdev_shmem / drm_fbdev_ttm / drm_fbdev_generic / drm_fb_helper / drm_fb_dma_helper**: fbdev compat shims
- **drm_client / drm_client_event / drm_client_modeset / drm_client_sysrq**: in-kernel DRM client (splash, sysrq emergency display reset)
- **drm_managed**: drmm_* devres-style lifecycle helpers

### Memory managers

- **TTM** (`ttm/`): VRAM-aware allocator + shmem swap-out + per-driver resource managers; consumed by amdgpu/i915/xe/nouveau/radeon/qxl/virtio-gpu/vmwgfx
- **GEM-shmem helpers** (top-level): pure system-memory GEM for SoC GPUs without dedicated VRAM
- **GEM-DMA helpers** (top-level): physically-contiguous GEM for SoC display

### Scheduler

- **gpu_scheduler** (`scheduler/`): `drm_sched` — generic GPU job scheduler with per-entity priority queues + fence chaining

### Per-vendor sub-drivers (active maintenance set)

| Driver | Path | Coverage |
|---|---|---|
| **amdgpu** | `amd/amdgpu/` + `amd/display/` + `amd/include/` + `amd/pm/` + `amd/ras/` + `amd/acp/` | Vega → RDNA4, MI100 → MI300, Phoenix iGPU |
| **amdkfd** | `amd/amdkfd/` | ROCm compute (`/dev/kfd`) |
| **amdxcp** | `amd/amdxcp/` | MI300 cross-CCD partitioning |
| **i915** | `i915/` | Intel Gen2 → Gen12.5 |
| **xe** | `xe/` | Intel Xe2+ (Lunar Lake, Battlemage) |
| **nouveau** | `nouveau/` | NVIDIA NV04 → Ada (community) |
| **radeon** | `radeon/` | AMD R100 → Northern-Islands (legacy) |
| **virtio** | `virtio/` | virtio-gpu paravirt (2D + 3D + cross-domain) |
| **vmwgfx** | `vmwgfx/` | VMware SVGA |
| **qxl** | `qxl/` | QEMU Spice |
| **ast** | `ast/` | ASPEED AST BMC |
| **mgag200** | `mgag200/` | Matrox G200 BMC |
| **tiny/bochs** | `tiny/bochs.c` | Bochs / qemu std-vga |
| **tiny/cirrus** | `tiny/cirrus.c` | Cirrus GD5446 (qemu legacy) |
| **gud** | `gud/` | Generic USB Display |
| **udl** | `udl/` | DisplayLink USB |
| **simpledrm** | `tiny/simpledrm.c` | EFI-stub-passed framebuffer (early boot) |

(ARM SoC drivers — `arm/`, `armada/`, `aspeed/`, `atmel-hlcdc/`, `bridge/`, `etnaviv/`, `exynos/`, `hisilicon/`, `imagination/`, `imx/`, `ingenic/`, `lima/`, `mediatek/`, `meson/`, `msm/`, `nouveau/` keep, `omapdrm/`, `panel/` mostly, `panfrost/`, `panthor/`, `pl111/`, `rcar-du/`, `renesas/`, `rockchip/`, `solomon/`, `sprd/`, `stm/`, `sti/`, `sun4i/`, `tegra/`, `tidss/`, `tilcdc/`, `tve200/`, `v3d/`, `vc4/`, `vboxvideo/`, `xlnx/`, `loongson/`, `logicvc/`, `mxsfb/` — most compile-gated off for v0; some preserved for ARM-on-x86_64-host nested-VM-display use).

### scope

This Tier-2 governs `/home/doll/linux-src/drivers/gpu/drm/` (~177 top-level files + dozens of subdirs), public headers `include/drm/*` (~80 headers), UAPI `include/uapi/drm/*` (per-driver UAPIs: drm.h + drm_mode.h + per-driver `<vendor>_drm.h` for amdgpu, i915, xe, nouveau, radeon, virtgpu, vmwgfx, qxl, etnaviv, exynos, lima, msm, panfrost, panthor, qaic, tegra, v3d, vc4, vgem, vivante, etc.).

### compatibility contract — outline

### `/dev/dri/card<N>` + `/dev/dri/renderD<N>` chardev IOCTLs

Per-driver UAPI ioctl numbers + struct layouts byte-identical so:
- libdrm consumes unchanged
- Mesa GLES/GL/Vulkan drivers consume unchanged
- Vulkan loader + per-vendor Vulkan ICDs (RADV, ANV, NVK, Tyr, V3DV, Turnip) consume unchanged
- Wayland compositors (gnome-shell, plasma-wayland, sway, hyprland, kwin) consume unchanged
- X11 (Xwayland + xorg-server with modesetting + xf86-video-amdgpu/intel/nouveau) consume unchanged
- KMS clients (kmscube, drm-howto, kmscon) consume unchanged

UAPI surface for amdgpu: `DRM_IOCTL_AMDGPU_*` (~30 ioctls). For i915: `DRM_IOCTL_I915_*` (~50 ioctls; many deprecated post-context-isolation). For xe: `DRM_IOCTL_XE_*` (~10 modern ioctls; designed clean). For nouveau: `DRM_IOCTL_NOUVEAU_*`. For virtgpu: `DRM_IOCTL_VIRTGPU_*`.

Generic DRM ioctls (`DRM_IOCTL_VERSION`, `_GET_UNIQUE`, `_GET_MAGIC`, `_GET_CLIENT`, `_GET_STATS`, `_GET_CAP`, `_SET_CLIENT_CAP`, `_AUTH_MAGIC`, `_DROP_MASTER`, `_SET_MASTER`, `_MODE_GETRESOURCES`, `_MODE_GETPLANE`, `_MODE_GETPLANERESOURCES`, `_MODE_GETCRTC`, `_MODE_SETCRTC`, `_MODE_GETCONNECTOR`, `_MODE_GETENCODER`, `_MODE_GETPROPERTY`, `_MODE_SETPROPERTY`, `_MODE_GETPROPBLOB`, `_MODE_GETFB`, `_MODE_ADDFB`, `_MODE_ADDFB2`, `_MODE_ADDFB2_MOD`, `_MODE_RMFB`, `_MODE_PAGE_FLIP`, `_MODE_DIRTYFB`, `_MODE_CREATE_DUMB`, `_MODE_MAP_DUMB`, `_MODE_DESTROY_DUMB`, `_MODE_OBJ_GETPROPERTIES`, `_MODE_OBJ_SETPROPERTY`, `_MODE_CURSOR`, `_MODE_CURSOR2`, `_MODE_GETGAMMA`, `_MODE_SETGAMMA`, `_MODE_ATOMIC`, `_MODE_CREATEPROPBLOB`, `_MODE_DESTROYPROPBLOB`, `_MODE_REVOKE_LEASE`, `_MODE_LIST_LESSEES`, `_MODE_GET_LEASE`, `_MODE_CREATE_LEASE`, `_GEM_CLOSE`, `_GEM_FLINK`, `_GEM_OPEN`, `_PRIME_HANDLE_TO_FD`, `_PRIME_FD_TO_HANDLE`, `_SYNCOBJ_CREATE`, `_SYNCOBJ_DESTROY`, `_SYNCOBJ_HANDLE_TO_FD`, `_SYNCOBJ_FD_TO_HANDLE`, `_SYNCOBJ_WAIT`, `_SYNCOBJ_RESET`, `_SYNCOBJ_SIGNAL`, `_SYNCOBJ_TIMELINE_*`, `_SYNCOBJ_QUERY`, `_SYNCOBJ_TRANSFER`) byte-identical.

### `/dev/kfd` AMD compute chardev

`KFD_IOC_*` ioctls byte-identical for ROCm runtime (HIP, OpenCL-on-AMD, ROCm-OpenCL, MIOpen, RCCL).

### sysfs surface

Per-card `/sys/class/drm/card<N>/`:
- `device/`, `drm_device`, `error`, `gt_*`, `gt/{gt0,gt1,...}/{cur_freq_mhz, max_freq_mhz, min_freq_mhz, ...}`, `power_management`, `vbt`, etc. (per-driver; i915/xe/amdgpu have rich sysfs)
- Per-connector `card<N>-<CONNECTOR>/{enabled, status, modes, dpms, edid, force, suspend, ...}`
- Per-CRTC `card<N>-<CRTC>/`

Per-renderer `/sys/class/drm/renderD<N>/{device,drm_device,...}`

Layout + content byte-identical so udevadm + intel_gpu_top + radeontop + nvtop + Sway DRM-info consume unchanged.

### debugfs

`/sys/kernel/debug/dri/<N>/` per-driver dump files (i915 has `i915_capabilities`, `i915_engine_info`, `i915_gem_objects`, `i915_runtime_pm_status`, `i915_wakeref`, `i915_dp_*`, `i915_audio_*`, `i915_power_domain_info`, `framebuffer`, `internal_clients`, `interrupt_info`, `i915_sseu_status`, `i915_panel_*`, `i915_drrs_status`, `i915_psr_*`, `i915_huc_load_status`, `i915_guc_*`, `i915_dsc_*`, `i915_lrc`, `i915_mei_pxp_*`; amdgpu has `amdgpu_pm_info`, `amdgpu_regs`, `amdgpu_firmware_info`, `gpu_recover`, `amdgpu_*`). Per-driver content; format byte-identical for in-tree consumers.

### Module params

- amdgpu: `amdgpu.dpm`, `amdgpu.audio`, `amdgpu.disp_priority`, `amdgpu.hw_i2c`, `amdgpu.pcie_gen2`, `amdgpu.msi`, `amdgpu.lockup_timeout`, `amdgpu.dpm`, `amdgpu.fw_load_type`, `amdgpu.aspm`, `amdgpu.runpm`, `amdgpu.bapm`, `amdgpu.deep_color`, `amdgpu.vm_size`, `amdgpu.vm_fragment_size`, `amdgpu.vm_block_size`, `amdgpu.vm_fault_stop`, `amdgpu.vm_debug`, `amdgpu.vm_update_mode`, `amdgpu.exp_hw_support`, `amdgpu.dc`, `amdgpu.dc_mask`, `amdgpu.dcdebugmask`, `amdgpu.no_evict`, `amdgpu.direct_gma_size`, `amdgpu.gart_size`, `amdgpu.gtt_size`, `amdgpu.queue_*`, `amdgpu.async_gfx_ring`, `amdgpu.recovery_method`, `amdgpu.bad_page_threshold`, `amdgpu.timeout_*`, `amdgpu.compute_multipipe`, `amdgpu.gpu_recovery`, `amdgpu.emu_mode`, `amdgpu.ras_*`, `amdgpu.ras_enable`, `amdgpu.ras_mask`, `amdgpu.smu_pptable_id`, `amdgpu.tcp_block_disable`, `amdgpu.use_xgmi_p2p`, `amdgpu.sched_jobs`, `amdgpu.sched_hw_submission`, `amdgpu.ppfeaturemask`, `amdgpu.forcelongtraining`, `amdgpu.cik_support`, `amdgpu.si_support`, `amdgpu.fw_load_type`, `amdgpu.virtual_display`, `amdgpu.audio`, `amdgpu.disp_priority`, `amdgpu.dummy_buffer_size`, `amdgpu.compute_priority`, `amdgpu.no_queue_eviction_on_vm_fault`, `amdgpu.svm_default_granularity`, ...
- i915: `i915.modeset`, `i915.enable_fbc`, `i915.semaphores`, `i915.enable_rc6`, `i915.enable_dc`, `i915.enable_psr`, `i915.disable_power_well`, `i915.enable_ips`, `i915.fastboot`, `i915.prefault_disable`, `i915.load_detect_test`, `i915.force_reset_modeset_test`, `i915.invert_brightness`, `i915.disable_display`, `i915.mmio_debug`, `i915.verbose_state_checks`, `i915.nuclear_pageflip`, `i915.edp_vswing`, `i915.enable_guc`, `i915.guc_log_level`, `i915.guc_firmware_path`, `i915.huc_firmware_path`, `i915.dmc_firmware_path`, `i915.gsc_firmware_path`, `i915.enable_dp_mst`, `i915.enable_psr2_sel_fetch`, `i915.disable_power_well`, `i915.enable_ips`, `i915.invert_brightness`, `i915.enable_pipefa_dropout`, `i915.fastboot`, `i915.alpha_support`, `i915.force_probe`, `i915.fastboot`, `i915.lvds_channel_mode`, `i915.panel_use_ssc`, `i915.vbt_sdvo_panel_type`, `i915.reset`, `i915.error_capture`, `i915.lvds_use_ssc`, `i915.modeset`, ...

Wire-format byte-identical (these param names appear in /etc/modprobe.d/ files at distros).

### EDID load + override

`/lib/firmware/edid/<file>.bin` loadable via `drm.edid_firmware=` cmdline. Wire format identical.

### `/sys/firmware/efi/efivars/Boot*` interaction

DRM doesn't write EFI vars but consumes EFI framebuffer pass-through (`simpledrm`) for early boot.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally; per-driver docs are large.)

| Tier-3 doc | Scope |
|---|---|
| `drivers/gpu/drm/core-drv.md` | `drm_drv.c` + `drm_dev*`: drm_driver registration, dev lifecycle |
| `drivers/gpu/drm/core-file-ioctl.md` | `drm_file.c` + `drm_ioctl.c`: per-fd state + ioctl dispatch |
| `drivers/gpu/drm/atomic.md` | `drm_atomic*`: atomic-modeset state machine + UAPI |
| `drivers/gpu/drm/kms-objects.md` | `drm_crtc.c` + `drm_plane.c` + `drm_connector.c` + `drm_encoder.c` + `drm_bridge.c` + `drm_panel.c`: KMS object hierarchy |
| `drivers/gpu/drm/kms-mode-property.md` | `drm_mode_object.c` + `drm_property.c` + `drm_blend.c` + `drm_color_mgmt.c` + `drm_colorop.c`: KMS property + blend + color pipeline |
| `drivers/gpu/drm/edid-displayid.md` | `drm_edid*` + `drm_displayid.c` + `drm_eld.c`: EDID + DisplayID parsing |
| `drivers/gpu/drm/gem-core.md` | `drm_gem.c` + GEM helpers |
| `drivers/gpu/drm/ttm.md` | `ttm/`: TTM allocator + resource managers |
| `drivers/gpu/drm/scheduler.md` | `scheduler/`: drm_sched |
| `drivers/gpu/drm/syncobj.md` | `drm_syncobj.c`: fence handle UAPI |
| `drivers/gpu/drm/buddy-mm-exec.md` | `drm_buddy.c` + `drm_mm.c` + `drm_exec.c` + `drm_suballoc.c` |
| `drivers/gpu/drm/dp-helper.md` | `display/drm_dp_*`: DisplayPort helper + MST + HDCP + DSC |
| `drivers/gpu/drm/lease.md` | `drm_lease.c` |
| `drivers/gpu/drm/vblank.md` | `drm_vblank.c` + `drm_vblank_work.c` |
| `drivers/gpu/drm/fbdev-shim.md` | `drm_fb_helper.c` + `drm_fbdev_*` + `drm_fb_dma_helper.c` |
| `drivers/gpu/drm/client.md` | `drm_client*` |
| `drivers/gpu/drm/amdgpu.md` | `amd/amdgpu/` + display/include/pm/ras: amdgpu universal driver |
| `drivers/gpu/drm/amd-display-dc.md` | `amd/display/`: AMD DC display engine |
| `drivers/gpu/drm/amdkfd.md` | `amd/amdkfd/`: ROCm `/dev/kfd` |
| `drivers/gpu/drm/amdxcp.md` | `amd/amdxcp/`: MI300 cross-CCD partitioning |
| `drivers/gpu/drm/i915-core.md` | `i915/i915_*`: i915 core + IOCTL surface |
| `drivers/gpu/drm/i915-gt.md` | `i915/gt/`: i915 GT — engine + workload-mgmt + GuC |
| `drivers/gpu/drm/i915-display.md` | `i915/display/`: i915 display engine |
| `drivers/gpu/drm/i915-gem.md` | `i915/gem/`: i915 GEM (per-driver) |
| `drivers/gpu/drm/xe.md` | `xe/`: Intel Xe2+ |
| `drivers/gpu/drm/nouveau.md` | `nouveau/`: community NVIDIA |
| `drivers/gpu/drm/radeon.md` | `radeon/`: legacy AMD pre-GCN |
| `drivers/gpu/drm/virtio-gpu.md` | `virtio/`: virtio-gpu paravirt |
| `drivers/gpu/drm/vmwgfx.md` | `vmwgfx/`: VMware SVGA |
| `drivers/gpu/drm/qxl.md` | `qxl/`: QEMU Spice |
| `drivers/gpu/drm/ast.md` | `ast/`: ASPEED AST |
| `drivers/gpu/drm/mgag200.md` | `mgag200/`: Matrox G200 |
| `drivers/gpu/drm/tiny-bochs-cirrus.md` | `tiny/bochs.c` + `tiny/cirrus.c` + `tiny/simpledrm.c` |
| `drivers/gpu/drm/gud-udl.md` | `gud/` + `udl/`: USB displays |

### compatibility outline (top-level)

- REQ-O1: All `/dev/dri/card<N>` + `/dev/dri/renderD<N>` ioctl wire format byte-identical (libdrm + Mesa + every Vulkan ICD consume unchanged).
- REQ-O2: `/dev/kfd` AMD compute IOCTLs byte-identical (ROCm consumes unchanged).
- REQ-O3: KMS atomic-modeset state machine + property semantics byte-identical (every Wayland compositor consumes unchanged).
- REQ-O4: EDID + DisplayID parsing produces identical mode lists vs upstream baseline for any reference monitor.
- REQ-O5: dma-buf import/export across drivers identical (V4L2 → DRM zero-copy works).
- REQ-O6: Per-driver sysfs + debugfs surface byte-identical for the v0 maintenance set.
- REQ-O7: Module params for amdgpu / i915 / xe / nouveau / radeon byte-identical (distro modprobe.d files work).
- REQ-O8: GEM/TTM allocator semantics + object lifetime byte-identical (cross-driver buffer sharing works).
- REQ-O9: drm_sched scheduling semantics + priority + dependency-fence chaining byte-identical (multi-context GPU consumers work).
- REQ-O10: Per-vendor command-submission ABIs byte-identical (Mesa command buffers work; per-driver UMD-KMD interface stable).
- REQ-O11: HDCP 1.4 + 2.2/2.3 link establishment works on supported hardware.
- REQ-O12: VRR / FreeSync / Adaptive-Sync property exposure identical.
- REQ-O13: TLA+ models declared at this Tier-2 (atomic-commit dependency ordering, drm_sched fence chain, GEM refcount + dma-buf attach race-freedom, TTM eviction + LRU, vblank event delivery, syncobj timeline-point monotonicity).
- REQ-O14: Verus/Creusot Layer-4 functional contracts on GEM mmap-offset alloc, drm_mm range-allocator, atomic-state object refcount.
- REQ-O15: Hardening: row-1 features applied per `00-security-principles.md`, with DRM-specific reinforcement (per-render-node CAP gating off — render nodes intentionally unprivileged; per-card CAP_SYS_ADMIN for master ioctls; userspace command-buffer command-validator on i915-pre-Gen11).

### acceptance criteria (top-level)

- [ ] AC-O1: `glxinfo -B` on amdgpu / i915 / xe shows correct vendor + renderer + Vulkan info matching upstream baseline. (covers REQ-O1, REQ-O10)
- [ ] AC-O2: `vulkaninfo` against RADV (amdgpu), ANV (i915), NVK (nouveau) reports correct device + extensions. (covers REQ-O1, REQ-O10)
- [ ] AC-O3: `kmscube` runs at native refresh rate on every supported display. (covers REQ-O3)
- [ ] AC-O4: GNOME Wayland session boots + plays compositor animations smoothly. (covers REQ-O3)
- [ ] AC-O5: Sway compositor runs on amdgpu reference HW; multi-monitor + VRR works. (covers REQ-O3, REQ-O12)
- [ ] AC-O6: Mesa CTS Vulkan conformance suite passes equivalent test count to upstream baseline on amdgpu RADV + i915 ANV. (covers REQ-O10)
- [ ] AC-O7: ROCm rocm-bandwidth-test on AMD MI200/MI300 reports HIP device + bandwidth matches upstream. (covers REQ-O2)
- [ ] AC-O8: V4L2 dma-buf import: GStreamer `v4l2src ! kmssink` zero-copies camera frames to display. (covers REQ-O5)
- [ ] AC-O9: HDCP 2.3 link to a HDCP-required HDMI monitor establishes; secure media playback succeeds. (covers REQ-O11)
- [ ] AC-O10: virtio-gpu guest in qemu boots Wayland Sway; 3D acceleration via virglrenderer works. (covers REQ-O10)
- [ ] AC-O11: kselftest `tools/testing/selftests/drm/` passes. (covers REQ-O13)
- [ ] AC-O12: drm/IGT (`igt-gpu-tools`) suite runs the basic-API tests on amdgpu + i915. (covers REQ-O13, REQ-O14)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/drm/atomic_commit.tla` | `drivers/gpu/drm/atomic.md` (proves: atomic-modeset commit dependency ordering — async commits properly ordered via vblank fences; CHECK + ATOMIC + NONBLOCK paths; concurrent commits on disjoint CRTCs progress in parallel; rollback on validation fail leaves no partial state) |
| `models/drm/sched_fence_chain.tla` | `drivers/gpu/drm/scheduler.md` (proves: drm_sched job-dependency chain — concurrent enqueue on N entities under scheduler thread; job-fence-signal triggers dependent jobs in FIFO order per priority; in-flight cancellation always completes pending fences with -ECANCELED) |
| `models/drm/gem_dmabuf.tla` | `drivers/gpu/drm/gem-core.md` (proves: GEM object refcount + dma-buf attach/detach + handle-from-fd + close race — concurrent gem_close + dma-buf release on cross-driver share never produces UAF or leaked refcount) |
| `models/drm/ttm_lru_evict.tla` | `drivers/gpu/drm/ttm.md` (proves: TTM LRU eviction — under memory pressure, BO-eviction + concurrent BO-pin + concurrent dma-buf import never evicts a pinned BO and never deadlocks LRU lock) |
| `models/drm/vblank_event.tla` | `drivers/gpu/drm/vblank.md` (proves: vblank event delivery — drm_pending_event queueing + per-fd event delivery + select/poll wakeup; missed-vblank counter monotonic; concurrent disable_vblank + enable_vblank converges to consistent enabled-counter) |
| `models/drm/syncobj_timeline.tla` | `drivers/gpu/drm/syncobj.md` (proves: syncobj timeline-point monotonicity — concurrent SIGNAL + WAIT on same timeline-point; per-syncobj seqno never goes backwards; cross-process syncobj-import preserves timeline) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `drivers/gpu/drm/core-file-ioctl.md` | drm_ioctl dispatch invariant: ioctl-cmd-table indexed by `_IOC_NR(cmd)` always within bounds; per-driver-overridden ioctls validated against `DRM_AUTH | DRM_RENDER_ALLOW | DRM_MASTER` flags |
| `drivers/gpu/drm/buddy-mm-exec.md` | `drm_mm_insert_node` post-condition: returned node satisfies size + alignment constraints; no overlap with existing nodes |
| `drivers/gpu/drm/atomic.md` | `drm_atomic_helper_commit` invariant: pre-commit + commit_tail + cleanup phases each invoked exactly once per commit; on -EAGAIN retry, prior state restored |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-drm_device + per-drm_file + per-gem_object + per-drm_master + per-syncobj refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-driver `drm_driver`, per-CRTC/plane/connector `*_funcs` + `*_helper_funcs`, per-bridge/panel ops `static const` | § Mandatory |
| **SIZE_OVERFLOW** | drm_mm range arithmetic + drm_buddy block-size + GEM/TTM page-count multiplications + framebuffer pitch×height use checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-driver drm_driver + per-driver ioctl-table + per-CRTC/plane/connector vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed GEM objects + TTM resources + dmabuf attachments cleared (carries cross-process pixel data + HDCP keys + DRM media stream content) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | per-driver firmware-load regions (i915 GuC/HuC/DMC, amdgpu SMU/PSP firmware) RX-only post-load | § Mandatory |

### Row-2 (LSM-stackable) features for DRM

LSM hooks called: file-LSM hooks on `/dev/dri/*` + `/sys/class/drm/.../*` writes + `/dev/kfd` open + ioctl. `security_drm_*` family (proposed for Rookery — upstream coverage minimal).

GR-RBAC adds:
- Per-role disallow `/dev/dri/card<N>` open (denies KMS-master access; only the compositor needs this).
- Per-role disallow `/dev/dri/renderD<N>` open (denies render-node access; user accounts can be allowed).
- Per-role disallow `/dev/kfd` open (denies ROCm compute access).
- Per-role disallow `DRM_IOCTL_SET_MASTER` / `_DROP_MASTER` (defends compositor-master from steal by other processes).
- Per-role disallow `DRM_IOCTL_MODE_CREATE_LEASE` (defends against split-screen lease abuse).
- Per-role audit of all DRM lease creation + drop.

### DRM-specific reinforcement

- **Render-node permission default**: `/dev/dri/renderD<N>` mode 0666 (anyone can render); `/dev/dri/card<N>` mode 0660 root:video (only compositor). Rookery preserves upstream defaults.
- **GEM `flink` global handle disabled by default**: prefer `prime_handle_to_fd` over global `flink` (legacy GEM handle that anyone could guess); CONFIG_DRM_LEGACY default-N.
- **i915 command-buffer parser**: pre-Gen11 needs userspace command parsing (un-trusted GPU command buffer sources kernel security risk); Rookery enforces parser-on for legacy gens; disable only via debugfs + CAP_SYS_ADMIN.
- **amdgpu firmware signature verification**: SMU + PSP + DMCUB firmwares signature-verified before load; bad signature → driver refuses init (defense against malicious firmware on disk).
- **HDCP key handling**: HDCP encryption keys never visible to userspace; only handle/cookie returned. Pre-Gen11 i915 PXP keys handled in dedicated PXP/MEI flow.
- **TTM eviction priority**: per-process TTM-eviction priority can't exceed parent process priority (defense against low-priority process stealing VRAM from high-priority compositor).
- **drm_sched per-process job-budget**: each `drm_file` capped at N concurrent jobs (default 1024); over-budget submit returns -EBUSY (defense against single client flooding scheduler).
- **dma-buf import LSM mediation**: cross-process dma-buf import goes through file-LSM hook on the source-fd; cross-VM (vfio) dma-buf import additionally validates IOMMU-binding consistency.
- **WRITEBACK CRTC permission**: writeback-CRTC fbo writeback (rendering CRTC output to memory) requires CAP_SYS_ADMIN (defense against keylogger via writeback capture of compositor output).
- **EDID firmware-override audit**: `drm.edid_firmware=` cmdline override logged at boot + any post-boot `force` write to `/sys/class/drm/.../edid_override` audited.

