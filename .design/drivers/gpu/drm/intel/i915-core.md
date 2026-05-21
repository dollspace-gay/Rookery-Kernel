# Tier-3: drivers/gpu/drm/i915/{i915_driver,i915_drm_client,i915_gem,i915_request,i915_active,i915_scheduler,i915_vma,i915_cmd_parser,i915_gem_evict,i915_gem_gtt,i915_perf,i915_pmu,i915_pci,i915_query,i915_drv}.c — Intel i915 core (every Intel iGPU + Arc discrete + Iris Xe)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpu/drm/00-overview.md
upstream-paths:
  - drivers/gpu/drm/i915/i915_driver.c                            (~1900 lines: drm_driver, pm_ops, module params, init/fini orchestration)
  - drivers/gpu/drm/i915/i915_pci.c                               (PCI ID table + per-platform info)
  - drivers/gpu/drm/i915/i915_drv.h                               (huge — types + caps + INTEL_INFO macros)
  - drivers/gpu/drm/i915/i915_drm_client.c                        (per-fd client tracking)
  - drivers/gpu/drm/i915/i915_gem.c                               (GEM core — mmap, busy, wait, set_domain)
  - drivers/gpu/drm/i915/i915_gem_evict.c                         (eviction policy)
  - drivers/gpu/drm/i915/i915_gem_gtt.c                           (GTT ABI; per-platform PT layouts live in gt/)
  - drivers/gpu/drm/i915/i915_gem_ww.c                            (wait-and-wound lock helpers for batched ww_mutex acquisition)
  - drivers/gpu/drm/i915/i915_request.c                           (per-request lifecycle on the scheduler)
  - drivers/gpu/drm/i915/i915_active.c                            (refcounted active-tracker for fences-on-objects)
  - drivers/gpu/drm/i915/i915_scheduler.c                         (sched node + priority + dependency tracking)
  - drivers/gpu/drm/i915/i915_vma.c                               (virtual memory area — per-VM BO mapping)
  - drivers/gpu/drm/i915/i915_vma_resource.c
  - drivers/gpu/drm/i915/i915_cmd_parser.c                        (~2000 lines: classic GEN6-9 batch parser for register allowlist enforcement)
  - drivers/gpu/drm/i915/i915_perf.c                              (perf counter subsystem — observability for HW)
  - drivers/gpu/drm/i915/i915_perf_oa_regs.h
  - drivers/gpu/drm/i915/i915_pmu.c                               (perf-events PMU integration)
  - drivers/gpu/drm/i915/i915_query.c                             (DRM_IOCTL_I915_QUERY)
  - drivers/gpu/drm/i915/i915_bo.c                                (BO core)
  - drivers/gpu/drm/i915/i915_dma_fence.c                         (dma-fence helpers)
  - drivers/gpu/drm/i915/i915_irq.c                               (top-level IRQ dispatch)
  - drivers/gpu/drm/i915/i915_mm.c                                (MM utilities — VMA range mgmt)
  - drivers/gpu/drm/i915/i915_module.c                            (module init)
  - drivers/gpu/drm/i915/gem/                                      (GEM subdir — per-purpose BO managers, ~30 files)
  - drivers/gpu/drm/i915/gt/                                       (GT subdir — per-engine: GFX, BLT, VCS, VECS, GSC; ~150 files)
  - drivers/gpu/drm/i915/display/                                  (display engine — DC equivalent, ~150 files, separate Tier-3)
  - drivers/gpu/drm/i915/pxp/                                      (PXP — Protected Content Protection)
  - drivers/gpu/drm/i915/selftests/                                (in-kernel tests)
  - drivers/gpu/drm/i915/intel_*.c                                 (per-platform implementations)
  - include/uapi/drm/i915_drm.h                                    (UAPI — ABI surface)
-->

## Summary

i915 is the Intel GPU driver covering every Intel graphics silicon from Sandy Bridge (2011) through Lunar Lake / Battlemage iGPU (2026) plus the discrete Arc Alchemist/Battlemage (DG1/DG2 + BMG) products. For most laptops and a majority of desktops shipping with Intel CPUs, i915 is the GPU driver responsible for the display, 3D rendering, video codec (via VCS engine + GSC), compute (via Iris Xe / Arc GPUs), and increasingly AI inference offload (Arc + Battlemage). On a typical 2026 Intel laptop (Lunar Lake, Arrow Lake H), i915 owns the iGPU; Intel's newer `xe` driver (drivers/gpu/drm/xe) is the planned successor for newer architectures but i915 continues to be the production driver for the installed base.

i915 is the second-largest single driver tree in upstream Linux at ~360,000 lines across 422 files. The structure:

- **Top-level (`i915_*.c`)** — driver lifecycle, GEM core, request scheduler, VM/VMA, cmd parser, perf, PMU, query, IRQ dispatch. The ~30K-line "kernel" of the driver.
- **`gem/`** — per-purpose BO managers (shmem, dmabuf, internal, lmem, mman, tiling, userptr, vm_bind, wait, busy, create, throttle).
- **`gt/`** — GT (Graphics Technology) — per-engine machinery: engine context, ring/lrc, execlists, GuC submission, reset, workarounds, RC6 power-gating, RPS frequency mgmt, TLB invalidate. ~150 files.
- **`display/`** — display engine (separate Tier-3): DPLL, DDI, plane, pipe, panel, audio, HDCP, HDMI, DP/eDP, MIPI DSI, color, gamma, atomic-state.
- **`pxp/`** — Protected Content Protection (Intel's HDCP-like + DRM playback).
- **Per-platform `intel_*.c`** — generation-specific tables, workarounds, golden registers.

This Tier-3 covers the ~30K-line core. Per-IP subsystems (GT engines, display, GuC, PXP, MEI bus integration) get their own iterations.

## Rust translation posture

i915 has the same scale challenge as amdgpu. The translation:

- **`i915-core` crate** holds the top-level driver lifecycle + GEM core + request scheduler + VMA. The `gt/`, `display/`, `pxp/`, `gem/` directories become sub-crates.
- **Per-generation typing.** Upstream's `IS_GEN(dev, X)` / `IS_GEN9` etc. macros become `GenInfo::{Gen6, Gen7, ..., Gen12, Xe2_LPG, Xe3_LPG, ...}` enum + per-gen trait impls. The `INTEL_INFO()` static table → typed `PlatformInfo` lookup at probe.
- **GEM objects.** `drm_i915_gem_object` becomes `I915Bo` with phase markers (`Allocated`, `Pinned`, `InUse`). Different BO backing types (`shmem`, `lmem`, `dmabuf`, `userptr`, `internal`) as enum variants.
- **VMA + VM.** `i915_vma` is a per-process per-BO mapping; `i915_address_space` is the per-process page table. Typed `Vma<Pinned/Unpinned>`.
- **Request scheduler.** `i915_request` is the unit of work; lives on a scheduler queue, has dependencies on prior requests, signals fences on completion. Typed `Request<Submitted/InFlight/Completed>`.
- **GuC submission.** Modern (Gen12+) uses GuC (Graphics microController) firmware-side scheduler — driver submits to GuC, GuC schedules engine work. Older (Gen8-11) uses execlists (driver-side scheduler that writes ELSP — Execlist Submit Port). Both modes typed.
- **Cmd parser.** Classic GEN6-9 had a kernel-side batch buffer parser that scrubbed user IBs against a register allowlist (because pre-PPGTT, batches could write privileged registers). For modern (Gen12+) full-PPGTT silicon, cmd parser is bypassed.
- **PXP (Protected Content).** Encrypted content session mgmt. PSP-equivalent (CSE/ME firmware) signs.
- **Wait-and-wound.** `ww_mutex` for batched BO reservation; Rookery models as RAII guard with explicit acquire-order.

The grsec/PaX section is mandatory. i915 has a very large UAPI surface (`i915_drm.h` has ~120 ioctls) + a long history of CVEs (CVE-2024-26603 in cmd parser, CVE-2023-1611 in GuC handling, etc.). The cmd parser specifically is a known bug-rich area.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct drm_i915_private` | top-level per-device state (huge) | `I915<Phase>` |
| `struct drm_i915_gem_object` | GEM BO | `I915Bo` |
| `struct i915_vma` | per-VM BO mapping | `Vma<S>` |
| `struct i915_address_space` | per-process VM | `Vm` |
| `struct i915_request` | per-request | `Request<S>` |
| `struct i915_sched_node` | scheduler node (request fragment + deps) | `SchedNode` |
| `struct intel_engine_cs` | per-engine (gt/) | `Engine` (gt sub-crate) |
| `struct intel_context` | per-(file, engine) ctx | `Ctx` |
| `i915_driver_probe(pdev, ent)` / `_remove(pdev)` | PCI probe / remove | `I915::probe` / `Drop` |
| `i915_driver_create(pdev, ent)` / `_destroy(...)` | per-device alloc | `I915::create` |
| `i915_driver_hw_probe(...)` / `_late_release(...)` | HW probe / late release | `I915::hw_probe` |
| `i915_driver_gt_probe(i915)` / `_gem_init(i915)` | per-GT and per-GEM init | `Gt::probe` / `Gem::init` |
| `i915_gem_object_create_*` (shmem, lmem, internal, ...) | BO create per-backing | `I915Bo::create_*` |
| `i915_gem_object_pin_pages(...)` / `_unpin_pages(...)` | page array pin/unpin | `I915Bo::pin_pages` |
| `i915_vma_instance(obj, vm, view)` | VMA lookup-or-create | `Vma::instance` |
| `i915_vma_pin(...)` / `_unpin(...)` / `_pin_ww(...)` | VMA pin | `Vma::pin` |
| `i915_request_create(ce)` / `_add(...)` / `_wait(...)` | request lifecycle | `Request::*` |
| `i915_scheduler_node_init(...)` / `_request_sched_node_init(...)` | scheduler init | `SchedNode::init` |
| `i915_cmd_parser_init(engine, ...)` / `_intel_engine_cmd_parser(...)` | classic cmd parser | `CmdParser::run` |
| `i915_gem_evict_for_node(vm, node, flags)` / `_evict_something(...)` | eviction policy | `Eviction::evict` |
| `i915_gem_evict_vm(vm)` | full-VM evict | `Eviction::evict_vm` |
| `i915_perf_init(i915)` / `_perf_open_ioctl(...)` / `_perf_release(...)` | perf counters | `Perf::*` |
| `i915_pmu_register(i915)` / `_unregister(...)` | perf-events PMU | `Pmu::register` |
| `i915_query_ioctl` | DRM_IOCTL_I915_QUERY | `Query::ioctl` |
| `i915_getparam_ioctl` / `i915_setparam_ioctl` | get/set param | `Param::get` / `set` |
| `i915_irq_handler(...)` / `_init_irq(...)` / `_disable_irq(...)` | top-level IRQ | `Irq::handler` / `_init` |
| `i915_gem_userptr_ioctl(...)` | userptr create | `Gem::userptr_ioctl` |
| `i915_gem_execbuffer2_ioctl(...)` | classic execbuffer | `Gem::execbuf2_ioctl` |
| `i915_gem_vm_bind_ioctl(...)` / `_vm_unbind_ioctl(...)` | modern VM-bind (Xe-aligned) | `Gem::vm_bind` / `_unbind` |
| `i915_gem_context_create_ioctl(...)` / `_destroy_ioctl(...)` | per-ctx | `Ctx::create` / `destroy` |
| `i915_gem_object_lock(...)` / `_unlock(...)` | per-object ww_mutex | `I915Bo::lock_ww` |
| `i915_gem_ww_ctx_init(...)` / `_unlock_all_and_done(...)` | wait-and-wound batched acquire | `WwCtx::*` |

## Compatibility contract

REQ-1: PCI ID table — every Intel GPU silicon: SNB/IVB/HSW/BDW/SKL/KBL/CFL/WHL/CML/CNL/ICL/EHL/JSL/TGL/RKL/ADL-P/ADL-S/RPL/MTL/LNL/ARL plus DG1/DG2 (Arc Alchemist) and BMG (Battlemage). Frozen against upstream `pciidlist[]`.

REQ-2: UAPI ABI — `include/uapi/drm/i915_drm.h` frozen against baseline; covers GEM, contexts, VM bind, query, getparam, execbuffer, perf, etc. ~120 ioctl numbers.

REQ-3: GEM domain model — BOs in domains: SHMEM (system RAM, swappable), LMEM (local VRAM on discrete), INTERNAL (driver-only), USERPTR (mmu-notifier-backed host mem), DMABUF.

REQ-4: VM types — Pre-Gen8 used a global GTT only; Gen8-9 added PPGTT (Per-Process Graphics Translation Table) optionally; Gen12+ requires full PPGTT (one VM per fd). Older silicon paths preserved for compat.

REQ-5: Request scheduler — drm-sched integrated; per-engine entity; priority + dependency tracking.

REQ-6: Engines — per-platform engine list: RCS (Render), BCS (Blitter — gone post-Gen11), VCS0/1 (Video Codec), VECS (Video Enhance), CCS (Compute, Gen12+), GSC (Graphics System Controller, Gen12+). Frozen against silicon spec.

REQ-7: Cmd parser (legacy) — for Gen6-9 pre-PPGTT, validates user batches against register allowlist (privileged registers refused). Bypassed for full-PPGTT silicon.

REQ-8: Wait-and-wound — multi-BO reservation uses ww_mutex to deadlock-avoid; user IB references many BOs, acquire-all-or-rollback.

REQ-9: VM bind (modern) — Xe-aligned VM_BIND/VM_UNBIND IOCTLs for batched BO-to-VA binding without explicit execbuffer.

REQ-10: Perf — OA (Observation Architecture) counters per-engine; per-context isolation; userspace tool integration (igt, perf, intel_gpu_top).

REQ-11: PMU — perf-events PMU exposing engine-busy time, frequency, RC6 residency.

REQ-12: PXP (Protected Content) — sessions over the CSE/ME firmware; HDCP key mgmt; encrypted output BO.

REQ-13: GuC firmware (Gen12+) — required for execution on Gen12+; HuC firmware (HEVC decode auth) optionally used.

REQ-14: Eviction — LRU per-VM eviction when LMEM/GTT pressure; swap-out to system RAM via TTM-like helpers.

REQ-15: Hot-plug — DG1/DG2 discrete cards support unplug via PCI hot-plug; driver gracefully tears down.

## Acceptance Criteria

- [ ] AC-1: Intel iGPU (Tiger Lake — Iris Xe) probes; i915 init completes; DRM device `/dev/dri/card0` appears; `dmesg | grep i915` matches upstream.
- [ ] AC-2: Mesa Iris (OpenGL/Vulkan) on Iris Xe: glmark2 runs at expected fps; vkCTS pass rate matches upstream.
- [ ] AC-3: Compute via SYCL / Level-Zero on Iris Xe / Arc; benchmark suite runs.
- [ ] AC-4: Video decode via VA-API (Mesa or Intel media driver): 4K HEVC/AV1 decode at >1x realtime.
- [ ] AC-5: Multiple compositor sessions: Wayland + Xwayland concurrent; no GPU hang.
- [ ] AC-6: Arc DG2 discrete: probes; LMEM allocator works; SR-IOV not supported (matches upstream).
- [ ] AC-7: GuC submission: per-engine GuC firmware loaded; engine work flows through GuC scheduler (`intel_gpu_top` shows GuC ctx).
- [ ] AC-8: Cmd parser on Gen9 (Kaby Lake): inject a batch with a privileged register write; cmd parser rejects.
- [ ] AC-9: VM bind: VM_BIND with batched map; subsequent execbuffer sees the mapping.
- [ ] AC-10: PXP smoke (Tiger Lake+): create PXP-protected session; encrypted output BO not CPU-mmap-able without keys.
- [ ] AC-11: Suspend/resume: S3 cycle preserves contexts + LMEM contents on discrete.
- [ ] AC-12: GPU recovery on hang: looped shader hangs the engine; reset fires; subsequent work succeeds.

## Architecture

**Per-generation specialization.** `intel_device_info` carries per-platform caps: gen, num_pipes, num_engines, has_lmem, has_logical_rings, has_guc, requires_force_probe, etc. Code branches on `GRAPHICS_VER(i915) >= N` rather than per-platform if-chains where possible. Rookery models as `GenInfo` enum with per-gen trait implementations; per-platform overrides where caps differ within a gen.

**GEM core.** BO types:
- **SHMEM** — system RAM-backed; pages allocated from shmem; swappable.
- **LMEM** — discrete card local VRAM; not swappable (eviction to SHMEM).
- **INTERNAL** — driver-only allocations.
- **USERPTR** — host process memory via mmu-notifier.
- **DMABUF** — imported from another driver.

Each is a typed Rust variant. Operations: create, mmap, pin_pages, unpin_pages, set_domain, busy, wait, destroy.

**VMA / VM.** Each open fd has a `i915_address_space` (VM). BOs are mapped into the VM at userspace-chosen VAs via execbuffer's reloc list (legacy) or VM_BIND (modern). VMA = per-VM-per-BO-per-view object; tracks pin state, address, view (full vs partial).

**Wait-and-wound batched BO reservation.** Single execbuffer touches N BOs; needs to acquire all their reservation locks. Naive: deadlock. `ww_mutex` solution: try to acquire each lock with `trylock`; on contention, drop all + restart with `wait`. The `i915_gem_ww_ctx` wraps this. Rookery: RAII guard `WwCtx` that owns acquired locks and unwinds on drop.

**Request lifecycle.** A submission creates an `i915_request` for each engine touched by the batch. Each request:
1. Created via `i915_request_create(ce)` where `ce` is an engine ctx.
2. Filled — batch buffer pointer, dependencies (other requests), per-request fences.
3. Submitted via `i915_request_add` → into the engine's scheduler queue.
4. Eventually run by the engine (GuC or execlists).
5. Completion signals the request's fence.

Rookery's `Request<S>` typestate: `Created → Submitted → InFlight → Completed`.

**Scheduler.** Per-engine sched queue; priority ordering; dependency tracking (`i915_sched_node` graph). Modern GuC mode delegates to GuC FW; execlists mode keeps the scheduler in driver and writes `ELSP` directly.

**Cmd parser (legacy).** On Gen6-9 pre-PPGTT silicon, user batches could read/write registers. To prevent privilege escalation, kernel parses each batch before submission, walking the cmd stream, validating each cmd's opcode + immediate register addresses against an allowlist. ~2000 lines of carefully-curated allowlist data. Bypassed when full-PPGTT (Gen12+) makes register access path-isolated by HW.

**Eviction.** When LMEM/GTT fills, evict LRU-style. Per-VM `vm->mm` (range tree) tracks pinned ranges; eviction picks unpinned BOs and unmaps them (swap to SHMEM or just unpin if SHMEM-backed). The `i915_gem_evict_for_node` finds a free range of N bytes; `_evict_vm` flushes the whole VM.

**Perf (OA).** Intel's per-engine performance counters. Userspace opens an OA stream, kernel programs OA registers + provides DMA-mapped report buffer; HW writes counter snapshots periodically; user mmap reads. Per-ctx isolation: only the requesting ctx's counters are returned.

**PMU.** Perf-events PMU exposing engine-busy, frequency, RC6 residency, etc. as standard perf-events.

**GuC submission (Gen12+).** GuC firmware-side scheduler. Driver populates GuC's submission tables, GuC schedules engine work. Eliminates per-engine driver-side scheduler (execlists). Adds GuC FW load + signature + crash recovery responsibilities.

**PXP (Protected Content).** Encrypted-content sessions over CSE/ME firmware. Setup: open PXP session via PXP TA (over MEI bus); HW-encrypted output BOs can't be CPU-mmap'd; HDCP integration in display side.

## Hardening

- All UAPI ioctl arg structs validated; per-platform feature gating (modern ioctls return -ENODEV on older silicon).
- Wait-and-wound has bounded retry; deadlock-on-rotation impossible.
- Request scheduler dependency graph has cycle detection.
- Cmd parser allowlist immutable; updates require kernel rebuild.
- VMA pin/unpin balanced; ref count saturating.
- Per-ctx scratch + LRC head/tail bounds checked.
- Eviction picks only unpinned BOs.
- GuC FW signature checked before activation.
- PXP session creation requires CAP_SYS_ADMIN.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — every i915 ioctl has a typed arg struct; copy_from_user bounded by sizeof(typed struct).
- **PAX_KERNEXEC** — `drm_i915_driver`, ioctl table, GEM ops, engine ops, IRQ tables, PMU ops, perf ops all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — per-syscall stack-rerandomize on every i915 ioctl entry.
- **PAX_REFCOUNT** — `drm_i915_private` refcount, per-BO refcount, per-VMA refcount, per-request refcount, per-ctx refcount saturating.
- **PAX_MEMORY_SANITIZE** — BO backing pages cleared on eviction-to-SHMEM (cross-process leak prevention). PXP-protected BOs always zero-on-free regardless of policy. GuC FW load buffers cleared after load.
- **PAX_UDEREF** — SMAP/SMEP on every ioctl + sysfs + debugfs + PMU.
- **PAX_RAP / kCFI** — drm_i915_driver, GEM ops table, engine ops, IRQ handler tables, GuC mailbox dispatch, PXP TA dispatch, perf-events PMU all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/dri/N/i915_*`) dumps include VM struct pointers, GuC firmware addresses, ring head/tail pointers; scrubbed for non-root.
- **GRKERNSEC_DMESG** — verbose i915 logs (per-request submit, GuC trace, eviction details) restricted from non-root.
- **DRM_AUTH + render-node split** — privileged ioctls (KMS) require DRM_MASTER; render ioctls (GEM, exec) accessible to authenticated render-node clients.
- **IOMMU mandatory for discrete (DG1/DG2/BMG)** — like amdgpu: GTT and userptr DMA from host pages; without IOMMU, DMA-attack surface. Integrated iGPUs use the CPU's main IOMMU.
- **Cmd parser allowlist closed** — register allowlist is a static const table; userspace cannot extend. Closes CVE-class where cmd parser missed a register and allowed privileged access.
- **Full-PPGTT mandatory on Gen12+** — refuses GGTT-only mode on Gen12+ silicon. GGTT-shared-with-display has been a long-standing leak path; full-PPGTT structurally closes it.
- **GuC FW signature mandatory** — GuC + HuC FW images PSP/CSE-signed; unsigned refused. LOCKDOWN_INTEGRITY_MAX disables FW updates.
- **PXP session CAP_SYS_ADMIN gated** — PXP grants access to encrypted content paths + HDCP keys; not for unprivileged.
- **PXP-protected BO mmap refused** — when a BO is part of a PXP session, the CPU cannot mmap it for read/write; only the GPU (with PXP key) can. Closes a path where an unprivileged process could read DRM-protected content.
- **Userptr mmu-notifier sync strict** — userptr BO holds an mmu-notifier; on host munmap, all in-flight GPU work on the BO is canceled before the notifier completes. Closes UAF-on-userptr race.
- **Per-ctx PMU scope** — perf OA opened per-ctx; non-root opens are restricted to the calling ctx's counters; cross-ctx snooping requires CAP_SYS_ADMIN.
- **Cmd parser disable refused via debugfs** — debugfs cannot disable cmd parser on affected silicon (would be a privilege-escalation primitive).
- **Request-priority for RT requires CAP_SYS_NICE** — same as amdgpu.
- **Self-test execution gated** — i915 has many in-kernel selftests; they require CAP_SYS_ADMIN to invoke (some are destructive).
- **DG2 LMEM clearing on alloc** — discrete LMEM is cleared on allocation (prevents cross-process leak of GPU-resident data when a process exits and another reclaims the LMEM region).
- **GuC reset-on-error rate-limit** — GuC-mediated engine resets rate-limited; > N in T seconds → permanent device offline.
- **PXP session timeout** — sessions auto-close after idle timeout; closes a "long-running PXP session retains keys" surface.

Per-doc rationale: i915 has 422 files / 360K LOC and a 120-ioctl UAPI — among the largest userspace-facing surfaces in the kernel. Historical CVE concentration in cmd parser (allowlist gaps), GuC handling (FW-driver protocol bugs), and execbuffer (user-supplied reloc handling) means the structural defenses must be solid. Rust translation's typed Request<S>/Vma<S> + WwCtx RAII close large bug classes. The full-PPGTT-mandatory-on-Gen12+ structurally closes the GGTT-leak class. PXP integration enforced by CSE TA + mmap-refuse policy closes DRM-content-read class. The cmd parser allowlist-as-immutable-const closes the parser-extension attack. Per-ctx PMU scope closes cross-ctx counter snooping. The IOMMU-mandatory gate on discrete cards closes the DMA-attack surface from imported BOs / userptr / GTT.

## Open Questions

- [ ] Q1: i915 vs xe (the planned successor) — Rookery has both to maintain. Strategy? Recommendation: i915 for installed-base + Gen12 production, xe for Xe2_LPG+ (Lunar Lake + Battlemage); user picks via Kconfig-equivalent.
- [ ] Q2: Cmd parser support — needed for Gen6-9 production hardware (still deployed). Keep the ~2000-line allowlist as a const data file in Rust. Updates require explicit version bumps.
- [ ] Q3: Display subsystem — ~80K lines, very tightly coupled to atomic-modesetting. Separate Tier-3 next iteration.
- [ ] Q4: GuC FW loading — depends on CSE/ME firmware availability; on systems without ME, GuC won't load. Gen12+ silicon mostly has ME; fallback behavior needs design.
- [ ] Q5: SR-IOV on data-center GPU (some Arc Pro variants) — limited deployment but exists. Defer to separate iteration.

## Verification

- **Kani SAFETY**: prove Request<Submitted>::wait cannot deadlock with the request's own dependency. Prove WwCtx acquired-locks set unwinds cleanly on Drop.
- **TLA+**: model the execbuffer wait-and-wound retry loop. Check progress (no infinite loop) + safety (no permanently-held lock).
- **Verus**: functional spec of `i915_cmd_parser` — for any user batch, either accepts (all cmds in allowlist) or rejects with -EINVAL; never lets a privileged-register-write through.
- **Kani+Verus**: invariant that every pinned VMA has a balancing unpin; every BO refcount reaches zero before kfree.
- **Integration**: full igt-gpu-tools test suite (multi-thousand-test compliance + stress suite); vkCTS Iris Vulkan; Mesa Iris OpenGL CTS; multi-Wayland-compositor stress; GuC submission stress; PXP-protected playback (Netflix Linux app or similar).
- **Fuzz**: execbuffer ioctl fuzzer (mutated reloc lists, BO handles, flags); GuC mailbox fuzzer; cmd-parser batch fuzzer.
- **Penetration**: unprivileged process attempts cmd-parser bypass via debugfs — refused. Unprivileged opens OA stream targeting another ctx — refused.

## Out of Scope

- `display/` — separate Tier-3 (next planned iteration for Intel)
- `gt/` engine details (per-engine workarounds, RC6, RPS) — separate Tier-3
- `gt/uc/` GuC + HuC + GSC firmware specifics — separate Tier-3
- `pxp/` PXP TA + MEI integration — separate Tier-3
- Per-generation workarounds — covered in `gt/intel_workarounds.c` — separate doc per major gen
- xe driver (`drivers/gpu/drm/xe/`) — separate Tier-3 (Xe2_LPG+ successor)
- Mesa Iris + Mesa ANV userspace drivers — out of kernel scope
- Intel media driver (`drivers/staging/...`) — out of scope
- TTM mid-layer — covered in drm core
