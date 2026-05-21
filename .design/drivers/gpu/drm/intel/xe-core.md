# Tier-3: drivers/gpu/drm/xe/{xe_pci,xe_device,xe_bo,xe_vm,xe_exec,xe_exec_queue,xe_gpu_scheduler,xe_dep_scheduler,xe_execlist,xe_drm_client,xe_query,xe_pm,xe_force_wake,xe_drm_ras,xe_configfs,xe_dma_buf,xe_devcoredump}.c — Intel xe (i915 successor — Xe2_LPG/Lunar Lake / Battlemage / Panther Lake / Xe3+ silicon)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpu/drm/00-overview.md
depends-on:
  - drivers/gpu/drm/drm-drv.md
  - drivers/gpu/drm/gem-core.md
  - drivers/gpu/drm/scheduler.md
sibling-but-distinct: drivers/gpu/drm/intel/i915-core.md
upstream-paths:
  - drivers/gpu/drm/xe/xe_pci.c                                  (~1360 lines: PCI ID table + per-platform info table + module init)
  - drivers/gpu/drm/xe/xe_device.c                               (~1430 lines: per-device init/fini + ioctl dispatch + drm_driver ops)
  - drivers/gpu/drm/xe/xe_bo.c                                   (BO core — TTM-backed, single resource model)
  - drivers/gpu/drm/xe/xe_bo_evict.c                             (eviction policy)
  - drivers/gpu/drm/xe/xe_dma_buf.c                              (dma-buf import/export)
  - drivers/gpu/drm/xe/xe_vm.c                                   (per-process VM — VM_BIND model)
  - drivers/gpu/drm/xe/xe_pt.c                                   (page-table walk + update)
  - drivers/gpu/drm/xe/xe_exec.c                                 (EXEC ioctl — submit work)
  - drivers/gpu/drm/xe/xe_exec_queue.c                           (per-engine execution queue per-ctx)
  - drivers/gpu/drm/xe/xe_gpu_scheduler.c                        (per-queue scheduler bridging to drm-sched)
  - drivers/gpu/drm/xe/xe_dep_scheduler.c                        (dependency scheduler)
  - drivers/gpu/drm/xe/xe_execlist.c                             (execlist backend for older silicon)
  - drivers/gpu/drm/xe/xe_drm_client.c                           (per-fd client tracking)
  - drivers/gpu/drm/xe/xe_query.c                                (DRM_IOCTL_XE_DEVICE_QUERY)
  - drivers/gpu/drm/xe/xe_pm.c                                   (D0/D3, runtime PM)
  - drivers/gpu/drm/xe/xe_force_wake.c                           (force-wake domains)
  - drivers/gpu/drm/xe/xe_drm_ras.c                              (RAS — per-engine UE/CE)
  - drivers/gpu/drm/xe/xe_devcoredump.c                          (coredump on hang)
  - drivers/gpu/drm/xe/xe_configfs.c                             (configfs for SR-IOV PF config)
  - drivers/gpu/drm/xe/xe_debugfs.c
  - drivers/gpu/drm/xe/xe_ggtt.c                                 (global GTT mgmt)
  - drivers/gpu/drm/xe/xe_drv.h
  - drivers/gpu/drm/xe/xe_device_types.h
  - drivers/gpu/drm/xe/xe_gt*.c                                  (GT subsystem — per-engine, separate Tier-3 family)
  - drivers/gpu/drm/xe/xe_guc*.c                                 (GuC firmware integration, separate Tier-3)
  - drivers/gpu/drm/xe/xe_gsc*.c                                 (GSC firmware integration, separate Tier-3)
  - drivers/gpu/drm/xe/xe_pxp*.c                                 (PXP — Protected Content Protection)
  - drivers/gpu/drm/xe/display/                                   (display engine, separate Tier-3)
  - include/uapi/drm/xe_drm.h                                     (UAPI — cleaner / smaller than i915)
-->

## Summary

xe is Intel's clean-rewrite GPU driver — the planned successor to i915 starting from Xe2_LPG (Lunar Lake iGPU, Battlemage discrete, Panther Lake) and going forward to Xe3+ silicon. xe was upstreamed in Linux 6.8 (2024) and made production-ready in subsequent releases. Architectural goals: cleaner UAPI than i915 (VM_BIND-first instead of execbuffer relocs), GuC-only submission (drop the dual GuC-vs-execlists code paths from i915, keep execlists only as a fallback for SR-IOV VF cases), better Mesa Vulkan / OpenCL integration, easier to maintain (one ABI shape for everything, no per-platform special cases like cmd parser).

xe ships **in addition to** i915, not as a replacement — Intel will keep i915 for the production-deployed installed base (Sandy Bridge through Meteor Lake / Raptor Lake) while xe owns Xe2_LPG forward. Rookery has both: i915 for the installed base, xe for new silicon.

xe in baseline 7.1.0-rc2: ~110000 LOC across 194 .c files. Significantly smaller than i915's 360K because of the clean-slate design and because many subsystems (display, GSC, MEI) are reused between i915 and xe via shared common code in `drivers/gpu/drm/i915/display/` (xe display borrows i915 display directly).

xe architecture:

- **Core** (covered here, ~10K lines) — driver lifecycle, BO, VM, EXEC, exec-queue, scheduler bridging, query, PM.
- **GT (`xe_gt*`)** — per-tile GT (Lunar Lake is a single-tile, Battlemage discrete is single-tile, future multi-tile MI-class designs may be multi-tile). Each GT has multiple engines.
- **GuC (`xe_guc*`)** — GuC firmware integration. GuC is *required* on xe (no execlists fallback for the primary path; execlists kept only for SR-IOV VFs where GuC is host-mediated).
- **GSC (`xe_gsc*`)** — Graphics System Controller; on-die firmware that runs PXP, RAS, OOB attestation, security primitives.
- **PXP (`xe_pxp*`)** — Protected Content Protection sessions.
- **Display (`display/`)** — borrowed from i915 display directory (shared code).
- **SR-IOV PF (`xe_sriov*`)** — host-side PF with GSC-mediated VF carve-out (modern Arc Pro + DG-class compute SKUs).

This Tier-3 covers the xe core (~10K lines).

## Rust translation posture

xe's clean-slate design is friendlier to Rust translation than i915. Strategy:

- **`xe-core` crate** holds driver lifecycle + BO + VM + EXEC + exec_queue + scheduler bridging.
- **Per-tile typing.** `Tile<T>` where `T` is a phantom tile-id; multi-tile silicon has multiple `Tile<T0>`/`Tile<T1>` instances. Forces explicit tile selection at compile time, prevents cross-tile state mix-ups.
- **VM_BIND model.** xe upstreamed the modern VM_BIND interface as the *only* way to manage VAs (no legacy execbuffer relocs). Translation: `Vm` exposes `bind_async(vma, op) -> Future<Result<...>>` only. Closes the entire reloc-list bug class structurally.
- **ExecQueue.** Per-(file, engine, ctx) typed handle; exec queue carries a drm-sched entity and a GuC ctx id; submission goes through it.
- **EXEC ioctl as typed.** Single `EXEC` ioctl takes the queue + BO list + batch addresses + sync fences in/out. Validation pipeline closed: `ExecRequest → ValidatedExec → ScheduledJob → Fence`.
- **BO as single TTM resource.** xe uses TTM directly (no I915-style per-backing-type variants); BO is one type with a TTM resource backing.
- **GuC-only submission path.** Eliminate the execlists vs GuC dual code. (Keep execlists in a `xe-execlist` sub-crate compiled only for SR-IOV VF builds.)
- **Force-wake domains** typed: `ForceWake<DomainKind>` with RAII guard.
- **PM phases typed.** `Power<D0/D3hot/D3cold>` transitions.

Grsec is mandatory. xe inherits much of i915's user-facing surface but with cleaner validation pipelines. The key structural defenses are stronger than i915 (no cmd parser needed — full PPGTT-only architecture; VM_BIND validation is closed).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct xe_device` | per-PCI-device state | `XeDev<Phase>` |
| `struct xe_tile` | per-tile state | `Tile<T>` |
| `struct xe_gt` | per-engine-cluster | `Gt` (gt sub-crate) |
| `struct xe_bo` | TTM-backed BO | `XeBo` |
| `struct xe_vm` | per-process VM | `Vm` |
| `struct xe_vma` | per-VM-per-BO mapping | `Vma` |
| `struct xe_exec_queue` | per-engine ctx for execution | `ExecQueue<EngType>` |
| `struct xe_sched_job` | drm-sched-bridged job | `SchedJob` |
| `xe_pci_probe(pdev, ent)` / `_remove(pdev)` | PCI probe/remove | `XeDev::probe` / `Drop` |
| `xe_device_create(pdev, ent)` / `_destroy(...)` | per-device alloc | `XeDev::create` |
| `xe_device_probe(xe)` | full bring-up | `XeDev::probe_full` |
| `xe_bo_create(...)` / `_create_locked(...)` / `_create_user(...)` | BO create | `XeBo::create_*` |
| `xe_bo_pin(...)` / `_unpin(...)` | pin/unpin | `XeBo::pin` |
| `xe_bo_evict(...)` / `_evict_all(...)` | eviction | `XeBo::evict` |
| `xe_vm_create(xe, flags)` / `_close(...)` / `_put(...)` | VM lifecycle | `Vm::create` / `Drop` |
| `xe_vm_bind_ioctl` / `_unbind_ioctl` | VM_BIND | `Vm::bind_ioctl` |
| `xe_pt_walk(...)` / `_pt_update(...)` | page-table walk / update | `Pt::walk` / `_update` |
| `xe_exec_queue_create_ioctl(...)` / `_destroy_ioctl(...)` | per-queue ctx | `ExecQueue::create` / `destroy` |
| `xe_exec_ioctl(dev, data, file)` | EXEC entry | `Exec::ioctl` |
| `xe_sched_job_create(...)` / `_arm(...)` / `_push(...)` | drm-sched bridging | `SchedJob::*` |
| `xe_exec_queue_get_property_ioctl(...)` / `_set_property_ioctl(...)` | ctx priority + persistence | `ExecQueue::prop` |
| `xe_query_ioctl` | DRM_IOCTL_XE_DEVICE_QUERY | `Query::ioctl` |
| `xe_wait_user_fence_ioctl` | user-fence wait | `WaitUserFence::ioctl` |
| `xe_pm_init(xe)` / `_runtime_suspend(...)` / `_runtime_resume(...)` | PM | `Pm::*` |
| `xe_force_wake_get(...)` / `_put(...)` | per-domain wake | `ForceWake::get` / `put` |
| `xe_devcoredump(...)` | hang coredump capture | `Coredump::capture` |
| `xe_drm_ras_*` | RAS hookups | `XeRas::*` |
| `xe_configfs_*` | SR-IOV configfs | `Configfs::*` |

## Compatibility contract

REQ-1: PCI ID table — Xe2_LPG (Lunar Lake iGPU), Xe2_HPG (Battlemage discrete BMG-G10/G21/G31), Panther Lake iGPU (Xe3), Xe3+ future silicon. Frozen against `xe_pci_id_table[]`.

REQ-2: UAPI ABI — `include/uapi/drm/xe_drm.h`. Frozen. ~25 ioctls — much cleaner than i915's 120.

REQ-3: VM_BIND model — VM_BIND is the only way to manage VAs. Operations: BIND, UNBIND, PREFETCH. Each is asynchronous and emits a fence. No legacy reloc lists.

REQ-4: BO model — single `xe_bo` type backed by TTM resource. Placement specified at create time (system memory, VRAM, dma-buf import).

REQ-5: ExecQueue — per-(file, engine, ctx) execution queue. Multiple queues per fd. Each holds a drm-sched entity + GuC ctx id.

REQ-6: EXEC ioctl — single ioctl: queue handle + BO list (implicit via VMA bindings) + batch addresses + sync-in fences + sync-out fence handle. Synchronous validation; asynchronous execution.

REQ-7: Sync fences — DRM syncobj (timeline + binary) supported for sync-in / sync-out. user-fence (memory-write-when-done) also supported.

REQ-8: GuC-only on primary path — submission goes through GuC. Execlists path only built for SR-IOV VFs where GuC is host-mediated.

REQ-9: Multi-tile — multi-tile silicon (e.g. some BMG-G31 variants, future MI-class) has multiple GTs; ABI exposes per-tile engine lists.

REQ-10: Force-wake — domain-scoped wake (render, blitter, media, gsc, gt); RAII reference counting.

REQ-11: PM — D0/D3hot/D3cold; runtime PM with idle threshold; RC6 power-gating.

REQ-12: RAS — per-engine UE/CE counters; per-engine soft reset.

REQ-13: SR-IOV PF — host-side PF can carve out N VFs via configfs; per-VF resource allocations through GSC-TA.

REQ-14: PXP — Protected Content sessions over GSC; encrypted output BOs.

REQ-15: Coredump — on engine hang, capture engine state + GuC log + recent submission history into a userspace blob.

## Acceptance Criteria

- [ ] AC-1: Lunar Lake iGPU (Xe2_LPG) probes; xe init completes; DRM device `/dev/dri/card0` appears.
- [ ] AC-2: Mesa Vulkan ANV on Lunar Lake: vkCTS pass rate matches upstream.
- [ ] AC-3: Battlemage discrete (BMG-G21): boot; LMEM allocator works; PCIe Gen4/5 link operational.
- [ ] AC-4: VM_BIND smoke: VM_BIND batch of 1000 maps; subsequent EXEC sees all mappings.
- [ ] AC-5: EXEC perf: single-queue throughput matches upstream baseline (vkCmdDraw per-second rate).
- [ ] AC-6: GuC submission: per-engine GuC loaded; engine work flows through GuC scheduler.
- [ ] AC-7: SR-IOV: Battlemage Pro carves into 4 VFs; each presented to a KVM guest; per-VF rendering works.
- [ ] AC-8: PXP: encrypted-output BO not CPU-mmap-able without GSC keys.
- [ ] AC-9: GPU recovery on hang: looped shader hangs an engine; reset fires; coredump captured.
- [ ] AC-10: Multi-tile: future multi-tile silicon — both tiles enumerate; per-tile engine independent.
- [ ] AC-11: Runtime PM: idle for >100ms; runtime suspend fires; first frame after wake within budget.

## Architecture

**Cleaner GEM than i915.** Single `xe_bo` type, single TTM backing. Placement (system, VRAM, dma-buf) decided at create. Eviction policy in `xe_bo_evict.c`. No per-backing-type variants like i915's SHMEM/LMEM/INTERNAL split.

**VM_BIND-first.** xe uses VM_BIND as the only address-space management primitive. UAPI exposes BIND/UNBIND/PREFETCH operations on a VM; each runs asynchronously through a bind-queue and emits a fence on completion. Subsequent EXEC submissions implicitly use the latest VM state. There's no legacy execbuffer-with-reloc-list path. This is the headline architectural difference from i915 and a major Rust-translation simplification.

**ExecQueue per-(file, engine, ctx).** Each open fd creates exec queues per engine it wants to drive. ExecQueue holds:
- drm-sched entity
- GuC ctx id (for GuC submission)
- Priority
- Persistence flag (whether the queue survives fd close)

EXEC ioctl targets a specific exec queue. Multiple queues per engine allowed; GuC schedules between them.

**EXEC ioctl pipeline.** EXEC ioctl:
1. Validate args (queue handle, batch addresses, sync in/out).
2. Resolve sync-in fences (wait or attach as deps).
3. Allocate `xe_sched_job`.
4. Push to drm-sched.
5. Return sync-out fence handle.

Pipeline is closed; ValidatedExec only constructible from validate. Closes the validation-skip bug class.

**Page-table walk.** `xe_pt.c` walks the VM's page tables for VM_BIND operations. Updates emitted as batch buffer operations to be executed by the engine (SDMA-style page-table writes), or via direct CPU writes for the GGTT.

**GuC submission.** Per-engine GuC ctx; driver submits via the GuC mailbox. GuC handles per-engine scheduling, preemption, context switching. Driver primarily maintains state coherency between drm-sched view and GuC view.

**Force-wake domains.** Some Intel HW reg accesses require explicit power-domain wake before they're valid. xe partitions into domains: render, media, blitter, gsc, gt. RAII handle `xe_force_wake_get(...)` reference-counts; `_put` releases.

**PM hierarchy.** Runtime PM: when no exec queues active for `pm_idle_ms`, runtime-suspend the device. D3cold supported on discrete. PCI L1 substates negotiated.

**Coredump.** On engine hang, capture: engine register state, recent submission history, GuC log, BO list. Stored as a binary blob accessible via `/sys/.../coredump`.

**SR-IOV PF.** Host-side PF reserves resources for VFs via GSC-TA. Configfs interface (`xe_configfs.c`) lets admin configure VF counts and resource quotas. Each VF is a separate PCI function; runs the same xe driver as a VF.

## Hardening

- VM_BIND validation closed; ValidatedBind type only constructible from validate.
- EXEC sync-in fence count bounded.
- BO size validated against TTM caps.
- ExecQueue count per fd bounded.
- GuC mailbox cmd timeout enforced.
- Coredump size capped.
- Force-wake reference count saturating.
- VM PT memory capped per VM.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — every xe ioctl has a typed arg struct; bounded `copy_from_user`. VM_BIND batches have per-op size cap.
- **PAX_KERNEXEC** — `xe_drm_driver`, ioctl table, GEM ops, GuC dispatch, GSC dispatch, PXP TA dispatch, RAS callbacks all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — per-syscall stack-rerandomize on every xe ioctl entry.
- **PAX_REFCOUNT** — XeDev refcount, per-BO refcount, per-VM refcount, per-ExecQueue refcount, per-SchedJob refcount, ForceWake reference count all saturating.
- **PAX_MEMORY_SANITIZE** — VRAM/LMEM cleared on alloc (LunarLake/Battlemage no cross-process leak). Userptr-imported pages held with mmu-notifier; on host-side munmap, GPU mappings invalidated synchronously. GSC cmd buffers cleared after submit. PXP session-state BOs cleared at session end.
- **PAX_UDEREF** — SMAP/SMEP on every ioctl + sysfs + configfs + debugfs.
- **PAX_RAP / kCFI** — drm_driver ops, ioctl table, GuC mailbox dispatch, GSC TA dispatch, PXP TA dispatch, RAS callbacks all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/dri/N/xe_*`) scrubbed; reveals VM struct pointers, GuC ctx ids, exec queue addresses.
- **GRKERNSEC_DMESG** — verbose xe logs (per-EXEC trace, GuC submission detail, VM_BIND ops) restricted from non-root.
- **DRM_AUTH + render-node split** — KMS ops require master; render ops accept authenticated render-node clients.
- **IOMMU mandatory for discrete (BMG)** — like amdgpu / i915 discrete; refuse probe without per-device IOMMU isolation.
- **VM_BIND closed validation** — typed `ValidatedBind` only constructible from validate. Closes whole class of "skip validation on error path" bugs (i915 had several CVEs in this pattern).
- **No cmd parser needed** — xe is full-PPGTT-only; cmd parser is structurally unnecessary. Closes the legacy cmd-parser-bug class entirely.
- **GuC firmware signature mandatory** — every GuC + GSC + HuC FW image signed; unsigned refused; LOCKDOWN_INTEGRITY_MAX disables FW updates.
- **GSC session strict** — every GSC TA invocation through typed mailbox; ABI versioning checked.
- **PXP CAP_SYS_ADMIN gated** — PXP session creation requires CAP_SYS_ADMIN; PXP-protected BOs un-mmap-able without GSC key.
- **Userptr mmu-notifier sync strict** — like i915, but stronger: typed VMA holds a `MmuNotifier` registration that's released only at BO destroy.
- **EXEC validation closed** — ValidatedExec only constructible from validate; cannot reach scheduling without validation.
- **GuC reset rate-limit** — GuC-initiated engine resets rate-limited; > N in T seconds → device offline.
- **Coredump access gated** — coredump may contain in-flight workload data + GuC log; CAP_SYS_ADMIN required.
- **SR-IOV PF configfs CAP_SYS_ADMIN** — VF config changes require CAP_SYS_ADMIN; PXP-aware so VF carve doesn't break protected sessions.
- **Per-ExecQueue PMU scope** — per-queue perf counters returnable to the queue owner only; cross-ctx snooping refused without CAP_SYS_ADMIN.
- **Force-wake closed RAII** — `ForceWake::get` returns a guard that auto-releases; an orphaned wake (forgotten release) is closed structurally.
- **PM state typed transitions** — `Power<D0>::to_d3hot` consumes self → `Power<D3hot>`. Wrong state transitions are compile errors.

Per-doc rationale: xe is the cleaner architectural successor. The structural defenses are stronger than i915 because of (1) full-PPGTT-only (no cmd parser needed, closes a bug class wholesale), (2) VM_BIND-only address space (no execbuffer-reloc complexity, closes a separate bug class), (3) GuC-only primary path (no dual-mode complexity), (4) cleaner UAPI (~25 ioctls vs 120 in i915). The Rust translation amplifies these defenses with typed ValidatedBind/ValidatedExec, typed Power<State>, typed ForceWake RAII, and per-tile typing. xe is the easier Rust port and the better long-term target for Intel silicon.

## Open Questions

- [ ] Q1: Should Rookery accept VF-execlists submission code path or refuse SR-IOV VF use entirely (host-only configurations)? Recommendation: refuse VF mode in v1; add later if needed.
- [ ] Q2: Display reused from i915 — does Rookery share the display sub-crate between i915 and xe, or build separate? Recommendation: share — display code is largely silicon-version-keyed, not driver-keyed.
- [ ] Q3: user-fence (memory-write-when-done) — adds complexity beyond syncobjs. Recommendation: support both; user-fence is the cleaner long-term primitive.
- [ ] Q4: GuC-only on primary path — if GuC FW unavailable (BIOS without CSE/ME), is the device unusable? Yes. Document the FW dependency clearly; refuse probe if GuC unloadable.
- [ ] Q5: VM_BIND async semantics — Rookery's Future-based bind needs careful integration with drm-sched fences. Plan for tokio-style runtime integration.

## Verification

- **Kani SAFETY**: prove ValidatedBind cannot be constructed without going through validate. Prove ForceWake::get/put balanced (refcount underflow impossible).
- **TLA+**: model the VM_BIND async pipeline with concurrent BIND/UNBIND on overlapping VAs. Check ordering is preserved + no UAF on overlapping ops.
- **Verus**: functional spec of the VM_BIND validation — for any input bind op, either rejects with -EINVAL or produces a ValidatedBind whose VA range is in the VM's address space.
- **Kani+Verus**: invariant that every active ExecQueue has a corresponding drm-sched entity; every BO mapped in a VM is tracked in the VM's VMA tree; no leaks at VM close.
- **Integration**: Mesa ANV Vulkan on Lunar Lake; Mesa Iris OpenGL; Level Zero compute on Battlemage; multi-Wayland-compositor; VM_BIND stress (1M bind/unbind cycles); GuC submission stress; PXP smoke; multi-tile (future silicon).
- **Fuzz**: EXEC ioctl fuzzer; VM_BIND ioctl fuzzer; query ioctl fuzzer; coredump ABI fuzzer.
- **Penetration**: unprivileged process attempts cross-ctx OA snooping — refused. Unprivileged attempts PXP session create — refused.

## Out of Scope

- xe `gt/` per-engine details — separate Tier-3
- xe GuC integration — separate Tier-3
- xe GSC integration — separate Tier-3
- xe PXP — separate Tier-3
- xe display (shared with i915) — covered in `i915-display.md` future iteration
- xe SR-IOV VF code path — out of scope per Q1
- i915 (the predecessor) — separate Tier-3, covered in `i915-core.md`
