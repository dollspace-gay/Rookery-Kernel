# Tier-3: drivers/gpu/drm/amd/amdgpu/{sdma_v6_0,amdgpu_sdma}.c — AMD RDNA3 / GFX11 SDMA engine (system DMA — page-table updates, BO copies, page-table walks)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpu/drm/00-overview.md
depends-on:
  - drivers/gpu/drm/amd/amdgpu-core.md
  - drivers/gpu/drm/amd/amdgpu-gfx-v11.md
upstream-paths:
  - drivers/gpu/drm/amd/amdgpu/sdma_v6_0.c                       (~1900 lines: SDMA 6.x IP — instance bring-up, microcode, ring init, IRQ, recover)
  - drivers/gpu/drm/amd/amdgpu/sdma_v6_0.h
  - drivers/gpu/drm/amd/amdgpu/sdma_v6_0_0_pkt_open.h            (SDMA 6 packet opcode definitions)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_sdma.c                     (~615 lines: cross-version SDMA helpers — instance abstraction, RAS hooks, FW load)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_sdma.h
  - include/asic_reg/sdma/sdma_6_0_*_offset.h                    (register offsets)
  - include/asic_reg/sdma/sdma_6_0_*_sh_mask.h                   (bitfield masks)
-->

## Summary

SDMA (System DMA) is the dedicated DMA engine on every modern AMD GPU. While the GFX engine executes user shaders, SDMA executes a separate command stream that performs: page-table updates (PD/PT writes for the per-VM page tables), BO copies (VRAM↔system-RAM moves during eviction, GTT setup, COPY ioctl), page-table walks for IB execution, fill/clear of VRAM regions, and small RAM-to-RAM DMA between BOs. SDMA is what makes GPU memory management asynchronous to the 3D pipeline — without SDMA, every page-table update would stall the GFX engine.

SDMA v6 is the SDMA generation paired with GFX11 (RDNA3): Navi 31/32/33, Phoenix1/2, Hawk Point, Strix Point. Same per-ASIC silicon-revision split as GFX11 — sdma_v6_0 for Navi 3x and sdma_v6_0_1/2/3/5 for various Phoenix/Strix revisions.

SDMA v6 silicon features:
- 2 SDMA engines on Navi 31, 1 on Navi 32/33 + Phoenix1/2.
- Each engine has 2 queues — one ring buffer + one page queue (high-priority for PT updates).
- Per-queue MMIO doorbell.
- Packet protocol distinct from GFX PM4 — SDMA opcodes (`SDMA_OP_NOP`, `SDMA_OP_COPY`, `SDMA_OP_WRITE`, `SDMA_OP_PTEPDE`, `SDMA_OP_FENCE`, `SDMA_OP_TRAP`, `SDMA_OP_POLL_REGMEM`, etc.).
- RAS counters integrated with amdgpu_ras.
- Power-gated when idle.

This Tier-3 covers ~2500 lines: `sdma_v6_0.c` IP block + `amdgpu_sdma.c` cross-version helpers.

## Rust translation posture

SDMA is structurally cleaner than GFX — fewer commands, no user-supplied shaders, deterministic packet protocol. The translation:

- **`amdgpu-sdma-v6` crate** implements `IpBlock` trait. Self-contained ASIC-register definitions, microcode loading, ring bring-up, IRQ handlers.
- **Typed SDMA packets.** Each opcode is a typed struct: `SdmaCopy { src: GpuAddr, dst: GpuAddr, count: u32 }`, `SdmaPtePde { pe: GpuAddr, value: u64, count: u32 }`, `SdmaFence { addr: GpuAddr, value: u64 }`. The packet builder serializes to the wire format with proper sub-opcode + flags.
- **Ring abstraction.** SDMA has one ring per engine + one page-queue per engine; typed `SdmaRing` vs `SdmaPageQ`.
- **GPU virtual address typing.** `GpuVa` is the GPU's virtual address (different from CPU virtual). SDMA commands take GpuVa for source/dst. Distinct type prevents host pointer → GpuVa confusion at compile time.
- **Page-table update batching.** The `amdgpu_vm.c` PT update path emits SDMA packets in batches; Rookery's `PtUpdateBatch` type collects updates and flushes via a single SDMA submission.
- **IB jump from GFX to SDMA.** Some workloads emit SDMA work from a GFX IB via INDIRECT_BUFFER → typed builder ensures the IB targets SDMA when used in that context.

Grsec is mandatory: SDMA can DMA anywhere in the GPU's address space — same DMA attack vector as GFX, but with simpler/more-deterministic packet protocol so easier to audit.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `sdma_v6_0_ip_funcs` | IpBlock for SDMA v6 | `SdmaV6IpBlock` |
| `sdma_v6_0_early_init` / `_sw_init` / `_hw_init` / `_sw_fini` / `_hw_fini` | init/fini | `SdmaV6IpBlock::*` |
| `sdma_v6_0_init_microcode` | FW load (PSP signed) | `Microcode::load_sdma_v6` |
| `sdma_v6_0_inst_start` / `_inst_stop` | per-instance engine start/stop | `SdmaInstance::start` / `stop` |
| `sdma_v6_0_ring_init` | per-ring config | `SdmaRing::init` |
| `sdma_v6_0_gfx_resume` / `_gfx_stop` | gfx-queue resume / stop | `SdmaRing::resume` |
| `sdma_v6_0_page_resume` / `_page_stop` | page-queue resume / stop | `SdmaPageQ::resume` |
| `sdma_v6_0_set_ring_funcs` / `_set_buffer_funcs` / `_set_vm_pte_funcs` / `_set_irq_funcs` | install op tables | `RingOps` / `BufOps` / `VmPteOps` / `IrqOps` |
| `sdma_v6_0_ring_emit_ib` | emit IB packet | `SdmaRing::emit_ib` |
| `sdma_v6_0_ring_emit_fence` | emit fence | `SdmaRing::emit_fence` |
| `sdma_v6_0_ring_emit_pipeline_sync` | wait for prior work | `SdmaRing::emit_sync` |
| `sdma_v6_0_ring_emit_vm_flush` | TLB flush | `SdmaRing::emit_vm_flush` |
| `sdma_v6_0_ring_emit_wreg` | privileged register write via ring | `SdmaRing::emit_wreg` |
| `sdma_v6_0_ring_emit_reg_wait` | register polling wait | `SdmaRing::emit_reg_wait` |
| `sdma_v6_0_emit_copy_buffer` | BO copy packet | `BufOps::copy_buffer` |
| `sdma_v6_0_emit_fill_buffer` | BO fill packet | `BufOps::fill_buffer` |
| `sdma_v6_0_vm_copy_pte` / `_vm_write_pte` / `_vm_set_pte_pde` | PT update emitters | `VmPteOps::*` |
| `sdma_v6_0_process_trap_irq` / `_process_illegal_inst_irq` | IRQ handlers | `IrqHandler::trap` / `_illegal_inst` |
| `sdma_v6_0_print_iv_desc` | error decode | `Iv::print` |
| `sdma_v6_0_soft_reset` | per-engine soft reset | `SdmaInstance::soft_reset` |
| `amdgpu_sdma_init_microcode_*` | cross-version FW loading helpers | `Microcode::*` |
| `amdgpu_sdma_ras_funcs` | RAS hooks | `SdmaRas::*` |

## Compatibility contract

REQ-1: ASIC silicon support — Navi 31 (sdma 6.0.0), Navi 32 (6.0.3), Navi 33 (6.0.2), Phoenix1 (6.0.1), Phoenix2 (6.0.4), Strix Point (6.0.5 + 6.1.x). Per-revision FW + register-bitfield offsets.

REQ-2: Microcode files — `amdgpu/sdma_6_0_0.bin` etc. PSP-signed. Frozen against `MODULE_FIRMWARE` declarations.

REQ-3: SDMA packet ABI — opcodes defined in `sdma_v6_0_0_pkt_open.h`. Frozen against silicon spec.

REQ-4: Per-engine ring count — 1 GFX ring + 1 page-queue per engine. 2 engines on Navi 31, 1 on Navi 32/33 + Phoenix.

REQ-5: Doorbell index — fixed allocation in the global doorbell space; per-engine + per-ring.

REQ-6: PT update path — `amdgpu_vm.c` emits PTE/PDE write packets to the page-queue (high-priority, dedicated). Page-queue cannot be preempted by GFX-ring work.

REQ-7: BO copy path — `amdgpu_ttm_copy_buffer` schedules an SDMA COPY packet via the drm-sched. Used at TTM eviction, GEM copy ioctl, and post-allocation fill-zero.

REQ-8: VM TLB flush — emitted at end of every IB before fence. SDMA-issued TLB flush is sufficient for SDMA-side VM state; GFX has its own TLB flush.

REQ-9: RAS — SDMA error counters integrated; per-engine UE/CE counts in `/sys/.../ras/sdma_*`.

REQ-10: Power-gating — when idle for `pg_idle` ms, SDMA engine clock-gated; transparently woken on next packet.

REQ-11: Soft reset — per-engine reset (without bouncing the rest of the GPU); used by amdgpu_ras for SDMA-scoped errors.

## Acceptance Criteria

- [ ] AC-1: RX 7900 XTX probes; SDMA v6 IP enumerates with 2 engines; microcode loads; ring + page-queue brought up.
- [ ] AC-2: BO copy: `vkCopyBuffer` or `glCopyBufferSubData` for a 1 GB BO completes via SDMA (verify via `amdgpu_top` or per-engine perf counter showing SDMA activity, not GFX).
- [ ] AC-3: PT updates async: while a GFX shader is running, allocate a new BO and `AMDGPU_GEM_VA` map it — PT update completes via SDMA without stalling GFX.
- [ ] AC-4: Page-queue high-priority: under heavy SDMA copy load, simultaneous PT update from a different process completes within the page-queue latency budget (sub-ms).
- [ ] AC-5: TLB flush correctness: BO remap to a different VA; subsequent IB sees the new mapping (proves TLB flush emitted + executed in correct order).
- [ ] AC-6: SDMA reset: trigger a hang (poll-regmem on a never-matching register); soft-reset fires; SDMA recovers; subsequent copies succeed.
- [ ] AC-7: RAS: induce a UE on SDMA via debugfs; counter increments; recovery scoped to SDMA engine only.
- [ ] AC-8: Multi-instance: on Navi 31 with 2 engines, run 2 parallel ttm-copy workloads; both engines active per amdgpu_top.

## Architecture

**Engine instance abstraction.** Each SDMA engine is an `amdgpu_sdma_instance` with: ring (gfx-class), page (page-queue), FW image, instance number, doorbell offset. `amdgpu_sdma.c` holds the cross-version instance helpers. v6 instances filled in by `sdma_v6_0_set_*` callbacks.

**Microcode.** Single FW per engine version (`sdma_6_0_0.bin` etc.). Loaded through PSP — driver hands the FW image to PSP TA, PSP verifies signature, writes to SDMA's FW SRAM. No driver-side direct SRAM write.

**Ring bring-up.** Per ring:
1. Allocate ring buffer in VRAM (typical 64 KB).
2. Init MQD with: ring base, ring size, rptr, wptr, doorbell index.
3. Write registers `SDMAx_QUEUEy_RB_*` for ring config.
4. Set doorbell ring-enable.
5. Ring is schedulable.

**Packet emission.** SDMA packets are little-endian DWORDs. Each packet header has `opcode + sub_opcode + flags`. Body varies per opcode. Examples:
- COPY: header + count + src/dst lo/hi DWORDs.
- WRITE: header + dst lo/hi + value lo/hi (×count).
- PTE/PDE: header + pe-address lo/hi + flags lo/hi + value lo/hi + count.
- FENCE: header + addr lo/hi + value lo/hi.
- POLL_REGMEM: header + poll-flags + addr + reference + mask + count.

Rookery emits via typed builders that serialize to the right wire format.

**PT update path.** `amdgpu_vm.c` collects pending PD/PT updates into a batch, allocates an SDMA IB, emits a sequence of `SDMA_OP_PTEPDE` packets, then a fence packet, submits to the page-queue. Page-queue is dedicated for PT updates — never carries user copy work.

**BO copy path.** TTM eviction or `AMDGPU_GEM_OP_COPY`:
1. Driver picks source domain (VRAM or system RAM via GTT IOMMU mapping).
2. Allocates an SDMA IB.
3. Emits `SDMA_OP_COPY` packets covering the BO's contiguous regions (a fragmented BO emits multiple packets).
4. Emits fence.
5. Submits to gfx-class ring.

**IRQ handling.** Per-instance trap IRQ for normal completion + illegal-instruction IRQ for malformed packets. Trap IRQ doesn't run per-packet (that's the doorbell completion path); it fires only on engine-level events like soft-stop done. Illegal-instruction fires on packet-decode error and triggers recovery.

**Recovery.** Soft reset per-engine: write `SDMAx_F32_SOFT_RESET`, poll for completion, then re-init the engine + re-arm ring. Used by amdgpu_ras for SDMA-scoped UEs.

## Hardening

- Packet emission via typed builders only; raw DWORD writes to ring not exposed.
- IB size bounded; oversized SDMA IB returns -EINVAL.
- PT update batch size bounded by ring size.
- VM TLB flush always emitted at end of IB (typed builder enforces).
- Per-engine FW image size validated.
- Soft-reset wait timeout (10ms); past timeout, escalate to full GPU reset.
- Doorbell index validated against driver-allocated set.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — no direct user surface; userspace can only feed SDMA via amdgpu_cs (which validates IBs in amdgpu_cs).
- **PAX_KERNEXEC** — IP block funcs, RingOps, BufOps, VmPteOps, IrqOps all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — per-syscall stack-rerandomize covered at amdgpu_cs entry.
- **PAX_REFCOUNT** — per-instance refcount, per-ring refcount saturating.
- **PAX_MEMORY_SANITIZE** — microcode load buffer cleared after PSP load. SDMA IB allocations from a pool that zeros on free (cmd contents include addresses + data that may be sensitive). Page-queue IB buffer zeroed at instance destroy.
- **PAX_UDEREF** — N/A direct user surface.
- **PAX_RAP / kCFI** — RingOps, BufOps, VmPteOps, IRQ dispatch all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs SDMA traces (instance state, ring pointers, FW image addr) scrubbed for non-root.
- **GRKERNSEC_DMESG** — SDMA verbose logs restricted from non-root.
- **Packet opcode allowlist** — SDMA packet emitters only support the documented opcode set; an attacker who could write to a ring buffer (none should — but defense in depth) could not introduce a new opcode that triggers undefined behavior.
- **SDMA can DMA anywhere — IOMMU mandatory** — SDMA's COPY/WRITE/FILL/PTE-PDE packets all take GPU-virtual addresses that the GPU MMU translates. *Without* IOMMU enforcement on the GPU's PCIe DMA side, a malformed IB could target arbitrary host memory. Same gate as amdgpu-core: refuse probe without IOMMU isolation.
- **VM TLB flush enforced** — typed builder ensures end-of-IB TLB flush. Closes a class where a packet sequence omitting TLB flush could see stale mappings.
- **PT update batch size capped** — per-batch packet count capped; closes a "user ioctl spawns giant batch that monopolizes the page-queue" DoS class.
- **Microcode signature mandatory** — PSP-signed; unsigned refused; LOCKDOWN_INTEGRITY_MAX disables FW updates.
- **Per-engine soft-reset rate-limit** — 3 per 60s, else escalate to full GPU reset.
- **Illegal-instruction IRQ → ctx kill** — when the engine reports an illegal-instruction (malformed packet), the ctx that submitted the offending IB is marked guilty + killed; no other ctxs affected.
- **POLL_REGMEM target allowlist** — SDMA `POLL_REGMEM` can wait on any register/memory address. Driver-emitted polls always target driver-controlled addresses; user IBs (CS path) have POLL_REGMEM scoped to user-mapped memory only (no MMIO register access from user IBs).
- **WRITE packet target allowlist** — same: driver-emitted WRITE goes to driver-controlled addresses; user IBs cannot use WRITE to target MMIO.
- **Pipeline-sync correctness** — typed `SdmaRing::emit_sync` ensures the pipeline-sync packet is emitted before any cross-ring dependency to prevent out-of-order observation.
- **Page-queue isolation** — page-queue only carries driver-emitted PT updates; user CSs cannot submit to the page-queue. Closes a path where user code could push fake PT updates from a CS IB.
- **RAS report scoped** — SDMA RAS report identifies engine + IB pointer; doesn't leak unrelated GPU state.

Per-doc rationale: SDMA is a smaller surface than GFX — fewer commands, no user shaders, deterministic packet protocol — but it has the same DMA-access power. The packet opcode allowlist + driver-only WRITE/POLL_REGMEM destinations close the "user IB does privileged DMA" class. Page-queue isolation closes the "user CS forges PT updates" class. The Rust translation's typed packet builders make malformed packets a compile error rather than runtime UB. Combined with the IOMMU-mandatory gate inherited from amdgpu-core, SDMA's DMA-attack surface is closed structurally.

## Open Questions

- [ ] Q1: Should user IBs ever be allowed to emit SDMA packets directly (via SDMA-to-SDMA INDIRECT_BUFFER), or always through driver-mediated paths? Upstream allows direct; closes a perf-optimization for some workloads. Recommendation: allow direct but validate the embedded packet sequence in the CS validator.
- [ ] Q2: Page-queue scheduler integration with drm-sched — page-queue is highest priority; should it be a separate drm-sched entity or always issued via the same scheduler? Recommendation: separate drm-sched-entity for page-queue.
- [ ] Q3: Phoenix1/2 (sdma 6.0.1/6.0.4) integration with PG — power-gating granularity differs from Navi; verify state preservation across PG cycles.

## Verification

- **Kani SAFETY**: prove SDMA packet builders only emit valid opcode/sub-opcode combinations. Prove `SdmaRing::emit_sync` is always emitted before a subsequent IB submission that depends on the prior one (typestate).
- **TLA+**: model the page-queue + gfx-ring scheduler; check that PT updates always complete before the GFX IB that depends on the new mappings starts.
- **Verus**: functional spec of the SDMA COPY emitter — for any (src_va, dst_va, count), produces a packet sequence whose execution copies exactly `count` bytes from `src_va` to `dst_va` (assuming GPU MMU translates both).
- **Kani+Verus**: invariant that the page-queue is only submitted to by driver-internal paths, never by user CS.
- **Integration**: large BO copy stress (10 GB BO copy on Navi 31); PT-update stress (1M BO maps in 60s, verify PT consistency); concurrent gfx + sdma workloads; SDMA soft-reset injection.
- **Fuzz**: SDMA IB fuzzer (mutate packet headers, lengths, addresses); driver must reject malformed or sanitize via packet-builder gating.
- **Penetration**: user CS attempts to emit POLL_REGMEM targeting a kernel-space register address — refused.

## Out of Scope

- Other SDMA versions (sdma_v4, v5, v7) — separate Tier-3 per version
- amdgpu_vm.c PT update logic — covered in amdgpu-core (caller of this driver's VmPteOps)
- TTM resource mgmt — covered in amdgpu-core
- GFX engine — covered in amdgpu-gfx-v11
- amdgpu_ras — covered in amdgpu-core
