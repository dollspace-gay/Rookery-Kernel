# Tier-3: drivers/gpu/drm/amd/amdgpu/{gfx_v11_0,gfx_v11_0_3,mes_v11_0,imu_v11_0,soc21}.c — AMD RDNA3 GFX engine (Radeon RX 7000 + Radeon Pro W7000 + Phoenix/Strix iGPU)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpu/drm/00-overview.md
depends-on:
  - drivers/gpu/drm/amd/amdgpu-core.md
upstream-paths:
  - drivers/gpu/drm/amd/amdgpu/gfx_v11_0.c                       (~7600 lines: GFX11 IP block — CP, RLC, MEC, golden registers, GRBM, microcode load)
  - drivers/gpu/drm/amd/amdgpu/gfx_v11_0.h
  - drivers/gpu/drm/amd/amdgpu/gfx_v11_0_3.c                     (~700 lines: GFX 11.0.3 variant — Navi 33, Phoenix1)
  - drivers/gpu/drm/amd/amdgpu/gfx_v11_0_3.h
  - drivers/gpu/drm/amd/amdgpu/gfx_v11_0_cleaner_shader.h        (compiled "cleaner shader" — wipes VGPRs/SGPRs/LDS between ctxs)
  - drivers/gpu/drm/amd/amdgpu/mes_v11_0.c                       (~1760 lines: MES — Micro-Engine Scheduler firmware loader + interface)
  - drivers/gpu/drm/amd/amdgpu/imu_v11_0.c                       (IMU — Intelligent Multimedia Unit boot)
  - drivers/gpu/drm/amd/amdgpu/soc21.c                            (SOC21 ASIC family wrapper — Navi3x + Phoenix + Strix)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_gfx.c / .h                  (cross-version GFX abstractions)
  - drivers/gpu/drm/amd/amdgpu/clearstate_gfx11_0.h               (CLEAR_STATE table)
  - include/asic_reg/gc/gc_11_0_*_offset.h, _sh_mask.h            (register offsets and bit definitions)
  - drivers/gpu/drm/amd/amdgpu/nvd.h                              (PM4 packet definitions for Navi family)
-->

## Summary

GFX11 is the 11th-generation AMD graphics IP — the 3D + compute engine inside RDNA3 silicon: Radeon RX 7900 XTX/XT/GRE (Navi 31), RX 7800 XT (Navi 32), RX 7700 XT/7600 (Navi 33), Radeon Pro W7900/W7800/W7700 (Navi 31 Pro variant), and the integrated GPU inside Ryzen 7040/8040/HX (Phoenix1/Phoenix2/Hawk Point) and Ryzen AI 300 (Strix Point). For these silicon, GFX11 is the engine that runs every Vulkan/OpenGL/DirectX-translated workload, every glsl/spv compute shader, every machine-learning workload that didn't take the dedicated XDNA/NPU path.

GFX11 introduced significant architectural changes from GFX10 (RDNA2):
- Bigger Compute Unit (CU) with 4× SIMD32 lanes per CU (vs 2× in GFX9/10) — doubled VALU width.
- New raytracing acceleration (BVH4 traversal in HW).
- DCC (Display-Compressible) on color targets enabled by default.
- New MES (Micro-Engine Scheduler) firmware-side scheduler replacing some kernel-side compute-queue management.
- Two-die Navi 31 has GCD (Graphics Compute Die) + 6× MCD (Memory Cache Die) — first AMD chiplet GPU.

The driver layer is the kernel-side controller for the engine: load microcode (CP-PFP, CP-ME, CP-MEC, RLC), bring up the rings (graphics, compute0..7, KIQ — Kernel Interface Queue), bring up MES, configure GRBM (Graphics Register Block Master), set up golden registers (per-revision tweaks), TDR handlers, suspend/resume.

This Tier-3 covers ~10000 lines of GFX11-specific code plus the cross-IP `amdgpu_gfx.c` abstractions.

## Rust translation posture

GFX11 is the largest IP block crate Rookery will produce. Strategy:

- **`amdgpu-gfx-v11` crate** implements the `IpBlock` trait from amdgpu-core. The crate is self-contained: ASIC register definitions, microcode handling, ring bring-up, golden-register sequences.
- **Microcode loading** is typed. CP (Command Processor) has multiple FW images (PFP, ME, MEC) each with its own signature; load through PSP TA (Trusted App) for signature verification.
- **Ring abstractions.** GFX11 has multiple ring types: GFX ring (graphics), 8 compute rings, KIQ (Kernel Interface Queue — used by driver to issue privileged ops). Each is a typed `Ring<GFX>`, `Ring<COMPUTE>`, `Ring<KIQ>`.
- **MES (Micro-Engine Scheduler).** MES is a firmware-side scheduler that owns compute queues. Driver interacts via a typed mailbox. Replaces some KIQ-managed work; queue map/unmap/preempt go through MES.
- **PM4 packets.** GFX uses the PM4 packet protocol — `bilge` typed PM4 packet types (`Packet3<OpCode>` for each opcode). Replaces the upstream macro-based PM4 emission with type-safe builders.
- **GRBM register access.** Per-SE (Shader Engine) / per-SA (Shader Array) register access goes through indexed-write-via-GRBM_GFX_INDEX. Typed `GrbmRegisterAccess` that scopes the per-SE/SA selection.
- **Golden registers.** Per-revision register-init sequences applied at hw_init. Typed `GoldenRegisterSet` per asic-revision.
- **Cleaner shader.** Compiled GFX11 binary that wipes VGPRs/SGPRs/LDS between contexts (closes information-leak across processes). Loaded as data at probe; invoked by drm-sched at job boundaries.
- **TDR (timeout-detection-and-recovery).** Per-ring TDR; on hang, identify the responsible CS, kill the ctx, GPU reset, re-arm. Typed state machine.

The grsec/PaX section is mandatory: GFX is the engine that executes user-supplied shader code. Shader-based information leaks (between processes, between privilege levels) are a known class.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `gfx_v11_0_ip_funcs` | IpBlock impl for GFX11 | `Gfx11IpBlock` |
| `gfx_v11_0_early_init` | early init — set caps, num CUs, etc. | `Gfx11IpBlock::early_init` |
| `gfx_v11_0_sw_init` | software init — allocate rings, CTX_REG, RLC table | `Gfx11IpBlock::sw_init` |
| `gfx_v11_0_hw_init` | bring up hardware — load FW, init RLC, bring up rings | `Gfx11IpBlock::hw_init` |
| `gfx_v11_0_late_init` | post-init — RAS hookup | `Gfx11IpBlock::late_init` |
| `gfx_v11_0_resume` / `_suspend` | suspend/resume | `Gfx11IpBlock::resume` / `_suspend` |
| `gfx_v11_0_set_ring_funcs` / `_set_irq_funcs` / `_set_rlc_funcs` / `_set_mqd_funcs` | per-ring/irq/RLC/MQD ops install | (typed `RingOps` trait impls) |
| `gfx_v11_0_set_cp_pfp_fw` / `_cp_me_fw` / `_cp_mec_fw` / `_rlc_fw` | microcode loader | `Microcode::load` |
| `gfx_v11_0_get_csb_size` / `_get_csb_buffer` | CLEAR_STATE buffer | `ClearState::get` |
| `gfx_v11_0_cp_resume` / `_cp_gfx_resume` / `_cp_compute_resume` / `_cp_async_gfx_ring_resume` | bring up CP rings | `CpResume::*` |
| `gfx_v11_0_kiq_resume` / `_kiq_set_resources` | KIQ bring-up | `Kiq::resume` |
| `gfx_v11_0_compute_mqd_init` / `_gfx_mqd_init` | per-queue MQD init | `Mqd::init` |
| `gfx_v11_0_ring_emit_*` (many — fence, ib, vm_flush, wreg, etc.) | PM4 packet emission per-ring | `Ring::emit_*` (typed) |
| `gfx_v11_0_priv_inst_irq` / `_priv_reg_irq` / `_squad_done_irq` | per-irq handlers | `IrqHandler::*` |
| `gfx_v11_0_setup_rb` / `_select_se_sh` / `_get_cu_info` | per-SE/RB/CU config | `CuInfo::query` |
| `gfx_v11_0_init_compute_vmid` / `_init_gds_vmid` | per-VMID GDS init | `VmidInit::*` |
| `gfx_v11_0_apply_cleaner_shader` | cleaner shader install + invoke | `CleanerShader::apply` |
| `gfx_v11_0_query_ras_*` / `_reset_ras_*` | RAS hooks | `Gfx11Ras::*` |
| `gfx_v11_0_unmap_done_irq` | preempt-done irq | `Preempt::on_done_irq` |
| `mes_v11_0_*` | MES IP block | `MesV11::*` |
| `mes_v11_0_load_microcode` / `_allocate_eop_buf` / `_init_data_buf` / `_misc_op` / `_query_status` | MES bring-up + op | `MesV11::load` / `_op` |
| `mes_v11_0_add_hw_queue` / `_remove_hw_queue` / `_unmap_queue` / `_map_queue` | MES queue mgmt | `MesV11::queue_*` |
| `imu_v11_0_*` | IMU boot | `ImuV11::*` |
| `soc21_*` | SOC21 ASIC wrapper (RDNA3 family) | `Soc21::*` |

## Compatibility contract

REQ-1: ASIC support — Navi 31 (gc_11_0_0), Navi 32 (gc_11_0_3), Navi 33 (gc_11_0_2), Phoenix1 (gc_11_0_1), Phoenix2 (gc_11_0_4), Strix Point (gc_11_5_*). Per-revision golden registers + microcode files.

REQ-2: Microcode files — `amdgpu/gc_11_0_0_pfp.bin`, `_me.bin`, `_mec.bin`, `_rlc.bin`, `_mes.bin`, `_mes1.bin` (+ variants per revision). All PSP-signed. Frozen against `MODULE_FIRMWARE` declarations.

REQ-3: Ring count — 1 GFX ring + 8 compute rings + 1 KIQ ring per pipe. Pipe/queue counts per ASIC.

REQ-4: PM4 packet ABI — IB packets, fence packets, VM flush packets, register writes, etc. matching GFX11 spec.

REQ-5: RLC (Run-List Controller) — handles power-gating per-CU dynamically; FW-driven.

REQ-6: MES — micro-engine scheduler; FW-side queue management; queue create/destroy via MES mailbox. KFD compute mode uses MES exclusively.

REQ-7: KIQ — driver-issued privileged ops (set register, write VRAM via INDIRECT_BUFFER, MAP_QUEUES, UNMAP_QUEUES). One KIQ per gfx-pipe.

REQ-8: GDS/GWS/OA — Global Data Share / Global Wave Sync / Ordered Append — small on-chip memory + sync resources per-VMID. Driver partitions per process.

REQ-9: Cleaner shader — compiled GFX11 binary loaded at boot; drm-sched invokes between job dispatches to wipe VGPRs/SGPRs/LDS (closes shader-side information leak).

REQ-10: TDR — per-ring timeout-detection-and-recovery; on hang, identify the responsible job, mark its ctx as guilty, GPU reset.

REQ-11: Compute partitioning — Navi 31 supports SE-level CU partitioning so multiple compute workloads can share the GPU; configured via amdkfd.

REQ-12: RAS — GFX11 has RAS counters for ECC events; integrate with amdgpu_ras.

REQ-13: Multi-die (Navi 31) — GCD + 6× MCD; driver sees as single device but RAS reports per-die.

## Acceptance Criteria

- [ ] AC-1: RX 7900 XTX probes; GFX11 IP block enumerates; microcode loads cleanly; KIQ ready.
- [ ] AC-2: vkCTS Vulkan compliance run via Mesa Radv on GFX11.
- [ ] AC-3: ROCm HIP `vector_add` sample on Navi 31 succeeds.
- [ ] AC-4: GPU recovery on shader hang: 1000-iteration infinite-loop shader injected via vkLoader; TDR fires within `tdr_timeout`; reset; subsequent dispatch succeeds.
- [ ] AC-5: Cleaner shader applied: write distinctive pattern in VGPR/SGPR from one ctx; switch ctx; verify next ctx reads zero.
- [ ] AC-6: MES queue map/unmap stress: 1000 KFD queues created and destroyed; no MES hang.
- [ ] AC-7: Multi-die RAS: induce a UE on one MCD; RAS reports per-die source; recovery scoped to affected die.
- [ ] AC-8: Suspend/resume across S3: full GFX state preserved (golden registers re-applied, microcode re-loaded, rings re-armed).
- [ ] AC-9: Phoenix iGPU: integrated graphics + simultaneous compute; both workloads share the engine.

## Architecture

**IP block init flow** (called from `amdgpu_device_ip_init`):
1. `early_init` — set caps (`adev->gfx.num_gfx_rings`, `num_compute_rings`, etc.); pick the per-revision golden register table.
2. `sw_init` — allocate rings (gfx + compute0..7 + kiq), allocate per-queue MQDs (Memory Queue Descriptors), allocate CSB (CLEAR_STATE Buffer), allocate RLC save/restore tables.
3. `hw_init` — load microcode through PSP, init RLC, set up GRBM, apply golden registers, set up CP rings, set up MES, set up KIQ, apply cleaner shader.
4. `late_init` — RAS hookup, set up TDR.

**Microcode loading.** Each FW image is loaded from `/lib/firmware/amdgpu/...bin`, sent through PSP TA for signature check, then PSP writes it into the GFX block's FW SRAM. Variants:
- CP-PFP (Prefetch Parser) — front-end of GFX ring.
- CP-ME (Microengine) — main packet processor.
- CP-MEC (Micro Engine for Compute) — compute-ring packet processor.
- RLC — runtime power-gating controller.
- MES + MES1 — micro-engine scheduler firmware.

**Ring bring-up.** GFX ring uses ROBR (Read-Only Buffer Register) addressing. Compute rings live in pipes (4 pipes × 2 queues each on most GFX11 ASICs). KIQ is on its own pipe. Bring-up sequence per-ring:
1. Allocate ring buffer (in VRAM).
2. Build MQD (Memory Queue Descriptor) with HQD (Hardware Queue Descriptor) layout: ring base, size, rptr, wptr, doorbell index.
3. Map MQD via KIQ MAP_QUEUES packet.
4. Ring is now schedulable.

**MES.** Replaces direct-KIQ queue management for compute. Driver sends an MES mailbox command (`ADD_QUEUE`, `REMOVE_QUEUE`, `UNMAP_QUEUE`, etc.) and MES updates the HW queue state. MES owns the compute-queue scheduling priorities.

**KIQ usage.** Driver issues:
- `SET_RESOURCES` at startup to claim pipes/queues.
- `MAP_QUEUES` to bring up new compute rings.
- `UNMAP_QUEUES` to drain a queue.
- Privileged register writes via INDIRECT_BUFFER.

**Cleaner shader.** A small GFX11 binary that fills all SIMD32 VGPRs + SGPRs + LDS with zero. drm-sched, between user-supplied jobs from different processes (different VMIDs), inserts a cleaner-shader dispatch. This closes shader-data leakage where one process's shader could read another's leftover register state.

**TDR.** drm-sched detects ring timeout. On timeout:
1. Stop the affected ring.
2. Identify the guilty job (the one being executed).
3. Mark the owning ctx with `AMDGPU_CTX_GUILTY`.
4. Trigger GPU reset (per-IP soft reset if possible, full reset if not).
5. Re-init affected rings.
6. Resume drm-sched.

**Multi-die (Navi 31).** GCD has GFX + compute, MCDs have L3 cache (Infinity Cache). RAS reports + power mgmt distinguishes per-die. Driver presents a single unified ASIC to userspace.

## Hardening

- Microcode size validated against expected per-version.
- PSP-mediated FW load enforces signature; unsigned FW refused.
- Ring MQD allocations bounded.
- KIQ command queue depth bounded.
- MES mailbox cmd timeout (5s); past timeout, MES-restart triggered.
- TDR rate-limit per ring (3 in 60s, then GPU declared offline).
- Cleaner shader invocation cannot be disabled without CAP_SYS_ADMIN + lockdown bypass (information-leak protection).
- All GRBM-indexed register accesses scoped via typed handle; cross-SE leak impossible.
- VMID allocation tracked; over-alloc returns -EBUSY.
- GDS/GWS/OA per-VMID allocations strict bounds.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — N/A directly; this driver is internal to amdgpu (userspace surface in amdgpu_cs).
- **PAX_KERNEXEC** — `gfx_v11_0_ip_funcs`, per-ring `amdgpu_ring_funcs`, RLC funcs, MQD funcs, IRQ handler tables, MES op tables all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — per-syscall stack-rerandomize at the amdgpu_cs entry point covers this driver's invocation paths.
- **PAX_REFCOUNT** — per-IP-block refcount, per-ring refcount, MES handle refcount use saturating refcounts.
- **PAX_MEMORY_SANITIZE** — microcode load buffer cleared after PSP load. RLC save/restore buffer cleared between contexts. Compute MQDs cleared on free. GDS/GWS/OA region cleared on VMID reassignment.
- **PAX_UDEREF** — N/A direct user surface.
- **PAX_RAP / kCFI** — IpBlock trait dispatch + per-ring `emit_*` calls + IRQ handler dispatch all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/dri/N/amdgpu_gfx_*`) scrubbed; reveals MQD pointers, ring base addresses.
- **GRKERNSEC_DMESG** — GFX trace verbose logs restricted from non-root.
- **CAP_SYS_RAWIO required for direct GFX register debugfs writes** — debugfs `gca_config_dump`, `gfx_clear_state`, etc.
- **Cleaner shader mandatory between ctxs** — the kernel inserts cleaner-shader dispatches between user ctx jobs whenever the next job's VMID differs from the previous. Disabling this requires CAP_SYS_ADMIN + LOCKDOWN bypass. Closes shader-side info-leak across processes.
- **Microcode signature mandatory** — CP-PFP/ME/MEC + RLC + MES all signature-checked by PSP TA. Unsigned microcode refused.
- **VMID isolation enforced by HW** — GFX accesses tagged with the issuing process's VMID; the GPU MMU enforces per-VMID page-table access. The driver guarantees a unique VMID per process; closes cross-process memory access via GFX.
- **GDS/GWS/OA partitioned per-VMID** — these on-chip resources are partitioned strictly; a process cannot read another's GDS slots.
- **TDR rate-limit** — 3 per-ring resets in 60s → permanent ring offline. Closes "compromised tenant submits hanging shader in loop" DoS class.
- **MES mailbox cmd allowlist** — only the known MES opcodes accepted; an attacker who could inject MES cmds (none should exist at the user level, but defense in depth) cannot trigger MES side effects beyond the spec.
- **Cleaner shader integrity** — the compiled cleaner shader binary is part of the driver and is cross-checked against an SHA-256 at module load (built into the binary). A tampered cleaner shader is refused.
- **Compute queue priority gated** — userspace can request a priority but the realtime/high priorities require CAP_SYS_NICE (matches the drm-sched policy).
- **GPU recovery initiated only by driver or KIQ** — user CS cannot directly trigger a GPU reset; reset events all funnel through the typed `AmdgpuDev::recover` path.
- **Per-process VMID allocator deterministic** — VMID assignment from a deterministic pool; closes a path where exhausting VMIDs could force a previously-deassigned VMID's stale state to leak across reuse (the sanitize-on-reassign covers the data side; deterministic allocation makes the reuse pattern auditable).
- **GFXOFF (power-gating) state restored cleanly** — entering/leaving GFXOFF preserves the right register state; closes a class where a malformed GFXOFF wake-sequence could expose stale state.

Per-doc rationale: GFX is the engine that executes user-supplied shader code. Shader information leaks (cross-process VGPR/SGPR/LDS), cross-VMID memory accesses, and TDR-recovery races are the bug classes. Cleaner shader mandatory between context switches closes the largest information-leak class structurally. VMID isolation enforced by HW + driver guarantees per-process VMID closes cross-process memory access. TDR rate-limit closes hang-attack DoS. Microcode signature mandatory closes the FW-substitution attack. Rust translation's typed Ring<EngineType> + typed PM4 builder + typed GRBM scoped access close additional bug classes (wrong-ring emit, wrong-SE indexed register access, cross-VMID mqd update). This is the first per-IP-block doc; the pattern established here (one crate per IP block + version) will be replicated for SDMA, VCN, MES variants, etc.

## Open Questions

- [ ] Q1: Cleaner shader — Rookery could ship a *more aggressive* cleaner shader that also clears scratch + GDS + GWS + OA partitions. Performance cost ~1µs per ctx switch. Recommendation: ship aggressive by default; opt-out via cmdline.
- [ ] Q2: MES SOS bypass — newer firmware allows the driver to bypass MES for some queue ops. Should Rookery use the bypass (more direct) or always go through MES (cleaner state machine)? Recommendation: through MES — defers complexity to the MES FW which is signed + audited by AMD.
- [ ] Q3: Multi-die RAS reporting — should per-MCD ECC counts be exposed in /sys/.../ras/ as separate files or aggregated? Recommendation: separate files for diagnosis + aggregated alias for monitoring.
- [ ] Q4: GFXOFF (power-gating) on integrated Phoenix — battery-life critical but introduces latency on the first dispatch after idle. Should the policy be tunable per workload? Recommendation: devlink param, default = balanced (auto-enter after 100ms idle).
- [ ] Q5: GFX11 has BVH4 raytracing acceleration; mostly used through Vulkan ray-query / NVIDIA-DXR-equivalent. Driver-side support is minimal (mostly compiler hands the right instructions). Any driver-side gating needed? Probably not.

## Verification

- **Kani SAFETY**: prove ring emit-fns can only be called on the correct ring type (typestate). Prove GRBM-indexed register access scopes correctly (no cross-SE leak via static analysis of the typed handle).
- **TLA+**: model TDR concurrent with new CS submission + MES queue map. Check that a TDR-in-progress drains all in-flight work cleanly; no new job starts during TDR.
- **Verus**: functional spec of the PM4 INDIRECT_BUFFER emitter — for any valid IB BO + size, produces a PM4 packet whose IB pointer/size are correctly little-endian-encoded.
- **Kani+Verus**: invariant that every MES ADD_QUEUE has a corresponding REMOVE_QUEUE before the MQD's BO is freed.
- **Integration**: vkCTS full pass; Mesa Radv complex shaders; ROCm HIP+OpenMP; TDR injection per ring; cleaner-shader effect verified by reading VGPRs across ctx; multi-die RAS injection on Navi 31.
- **Fuzz**: shader fuzz with rocBLAS workloads; CS-ioctl fuzz with malformed PM4 IBs (should refuse cleanly).
- **Penetration**: process A writes distinctive pattern into VGPRs via shader; process B's shader reads VGPRs and checks for the pattern; cleaner shader must zero before B reads.

## Out of Scope

- Other GFX IP versions (gfx_v9, v10, v12) — separate Tier-3 per version
- SDMA IP versions (sdma_v4/5/6/7) — separate Tier-3
- VCN / JPEG IP versions — separate Tier-3
- amdgpu_dm display — separate Tier-3
- SMU power-mgmt firmware — separate Tier-3
- amdkfd (compute interface) — separate Tier-3
- Shader compilation (Mesa Radv handles this in userspace)
- Per-ASIC golden register tables — listed here, full tables in include/asic_reg headers
- Cleaner shader source — assembly file `gfx_v11_0_cleaner_shader.asm` — not modified; binary loaded as data
