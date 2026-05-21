# Tier-3: drivers/gpu/drm/nouveau/{nouveau_drm,nouveau_abi16,nouveau_uvmm,nouveau_chan,nouveau_bo,nouveau_gem,nouveau_exec,nouveau_sched,nouveau_svm,nouveau_dmem,nouveau_fence,nouveau_vmm,nouveau_acpi,nouveau_bios,nouveau_pmops}.c + nvkm/ — Nouveau (NVIDIA open-source — Tesla/Fermi/Kepler/Maxwell/Pascal/Volta/Turing/Ampere/Ada/Blackwell via GSP firmware)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpu/drm/00-overview.md
upstream-paths:
  - drivers/gpu/drm/nouveau/nouveau_drm.c                       (~1550 lines: drm_driver, PCI probe/remove, runtime PM, switcheroo)
  - drivers/gpu/drm/nouveau/nouveau_abi16.c                     (~920 lines: legacy ABI16 ioctls for pre-r535 chips)
  - drivers/gpu/drm/nouveau/nouveau_uvmm.c                      (~2005 lines: per-process VM — modern VM_BIND-style)
  - drivers/gpu/drm/nouveau/nouveau_chan.c                      (channel — per-engine ctx)
  - drivers/gpu/drm/nouveau/nouveau_bo.c                        (BO core, TTM-backed)
  - drivers/gpu/drm/nouveau/nouveau_bo*.c                       (per-chip-class BO copy methods — bo0039, bo74c1, bo85b5, bo90b5, bo5039, bo9039, boa0b5)
  - drivers/gpu/drm/nouveau/nouveau_gem.c                       (GEM ioctls)
  - drivers/gpu/drm/nouveau/nouveau_exec.c                      (EXEC ioctl — modern, GSP-aware)
  - drivers/gpu/drm/nouveau/nouveau_sched.c                     (drm-sched bridging)
  - drivers/gpu/drm/nouveau/nouveau_svm.c                       (Shared Virtual Memory — unified-virtual address, HMM-backed)
  - drivers/gpu/drm/nouveau/nouveau_dmem.c                      (device memory — VRAM allocator backing SVM)
  - drivers/gpu/drm/nouveau/nouveau_fence.c                     (fence implementation)
  - drivers/gpu/drm/nouveau/nouveau_vmm.c                       (VM manager — coordinates uvmm + ABI16)
  - drivers/gpu/drm/nouveau/nouveau_acpi.c                      (ACPI integration — _DSM, Optimus hot-add)
  - drivers/gpu/drm/nouveau/nouveau_bios.c                      (VBIOS parse)
  - drivers/gpu/drm/nouveau/nouveau_pmops.c                     (PCI PM ops, runtime PM)
  - drivers/gpu/drm/nouveau/nouveau_connector.c, nouveau_display.c, ... (display side, separate Tier-3)
  - drivers/gpu/drm/nouveau/nvkm/                                (NVKM — Nouveau Kernel Module — per-subdev/engine implementations, ~120K lines)
  - drivers/gpu/drm/nouveau/nvkm/subdev/gsp/                     (GSP — GPU System Processor firmware interface, modern Turing+)
  - drivers/gpu/drm/nouveau/nvkm/engine/                         (engine implementations — gr, ce, dec, msvld, msenc, msppp, msvld, sec, etc.)
  - drivers/gpu/drm/nouveau/nvif/                                (NVIF — Nouveau Virtual Interface, type-safe internal API)
  - drivers/gpu/drm/nouveau/include/                             (NVKM internal headers)
  - include/uapi/drm/nouveau_drm.h                               (UAPI)
-->

## Summary

nouveau is the open-source NVIDIA GPU driver in upstream Linux. It supports **every NVIDIA GPU** from NV04 (1998) through Blackwell (2024) — Tesla (NV40+), Fermi, Kepler, Maxwell, Pascal, Volta, Turing, Ampere, Ada Lovelace, Blackwell. The driver has two distinct operating modes determined by the silicon generation:

- **Pre-Turing path (legacy direct register access)** — driver directly programs GPU registers + manages page tables + handles engine state. Reverse-engineered from observed NVIDIA proprietary driver behavior. Limited modern feature support; rough Vulkan; OpenGL works.
- **GSP firmware path (Turing+, since Linux 6.7 / 2024)** — driver talks to the GPU's on-die GSP (GPU System Processor) running NVIDIA-signed firmware (GSP-RM, the same RM = Resource Manager that the proprietary driver uses). Driver becomes a thin client: most GPU mgmt happens FW-side. This dramatically simplified nouveau for modern silicon and enabled real Vulkan + compute support on Turing/Ampere/Ada/Blackwell. The price: dependency on NVIDIA-signed FW (no open-FW path for these chips).

Together these two paths make nouveau the only fully-upstream NVIDIA driver. Mesa NVK is the Vulkan driver on top. NVIDIA's `nvidia.ko` proprietary driver still exists in parallel (commercial out-of-tree); nouveau is the open alternative.

The source tree is ~170,000 lines / 773 files. Structure:

- **`nouveau_*.c`** (~25K lines) — top-level drm_driver, DRM/GEM core, ioctl dispatch, channel + ctx mgmt, modern UAPI (uvmm/exec/sched).
- **`nvkm/`** (~120K lines) — NVKM (Nouveau Kernel Module), per-subdev + per-engine implementations covering every silicon generation. Each subdev (PMC, PRIV, FB, MMU, INSTMEM, FAULT, BUS, GPIO, I2C, BIOS, GSP, etc.) has per-chip-class .c files (gv100.c, ga102.c, ad102.c, gb202.c).
- **`nvif/`** — NVIF (Nouveau Virtual Interface), a type-safe internal API between nouveau (DRM-facing) and NVKM (HW-facing). Lets nouveau swap NVKM internals without touching DRM/UAPI code.
- **`display/`** — display engine (separate Tier-3).
- **GSP RM (`nvkm/subdev/gsp/rm/`)** — per-RM-version interface (r535, r570 etc.) — GSP firmware ABI tracking proprietary RM versions.

This Tier-3 covers the ~25K-line core. Per-subsystem (NVKM subdev/engine, GSP RM, display, SVM) get their own iterations.

## Rust translation posture

nouveau has the most challenging architecture: two distinct execution paths (legacy direct vs GSP-FW) covering 25+ years of silicon, with a per-chip-class subdev table that has hundreds of variants. Translation:

- **`nouveau-core` crate** holds the top-level drm_driver, UAPI ioctls, channel mgmt, GEM core.
- **`nvkm-core` crate** holds the NVKM framework (subdev/engine traits, per-chip dispatch).
- **Per-chip-class crates.** `nvkm-tu102`, `nvkm-ga102`, `nvkm-ad102`, `nvkm-gb202`, etc. each implementing the subdev + engine traits for that chip class.
- **Execution path typing.** `NouveauDev<Legacy>` vs `NouveauDev<Gsp>` distinct types. Methods that only make sense on GSP path (e.g. RM client mgmt) only exist on the GSP type. Pre-Turing silicon paths through the Legacy type.
- **GSP RPC.** GSP communication is RPC-style with typed cmd/resp pairs. RM ABI version is per-FW-image (r535, r570); typed `GspRm<R535>` vs `GspRm<R570>`.
- **NVIF as typed API.** NVIF's runtime-typed message ABI becomes Rust typed methods on `NvifClient`.
- **UVMM (modern VM_BIND).** Replaces ABI16 (legacy GEM). UVMM ops typed: `Bind`, `Unbind`, `Map`, `Unmap`. Drm-gpuvm framework used.
- **SVM/HMM integration.** GPU access to host process pages via mmu-notifier; typed `SvmRange<Active/Faulting>`.
- **Channel + fence.** Per-engine channel; typed `Channel<EngineType>`; fence per-channel.

The grsec/PaX section is mandatory. nouveau historically had a long CVE tail (CVE-2024-26695 SVM UAF, CVE-2023-3863 channel state machine race, multiple pre-GSP-era kernel-side parser bugs). The GSP path offloads complexity to the FW but adds the new attack surface of GSP-RPC parsing.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nouveau_drm` | per-device state | `NouveauDev<Path>` |
| `struct nouveau_channel` | per-engine ctx | `Channel<Engine>` |
| `struct nouveau_bo` | TTM-backed BO | `NouveauBo` |
| `struct nouveau_cli` | per-fd client | `Cli` |
| `struct nouveau_uvmm` | per-process modern VM | `Uvmm` |
| `struct nouveau_svmm` | SVM domain | `Svmm` |
| `struct nouveau_fence` | fence | `Fence` |
| `nouveau_drm_probe(pdev, ent)` / `_drm_remove(pdev)` | PCI probe/remove | `NouveauDev::probe` / `Drop` |
| `nouveau_drm_device_init(...)` / `_fini(...)` | per-device init/fini | `NouveauDev::init` |
| `nouveau_cli_init(drm, sname, cli)` / `_fini(...)` | per-fd client | `Cli::new` / `Drop` |
| `nouveau_channel_new(drm, device, runlist, engine, priv, vram, gart, chan)` | per-engine channel create | `Channel::create` |
| `nouveau_channel_del(chan)` | destroy | `Channel::Drop` |
| `nouveau_chan_wait(chan, busy)` | wait idle | `Channel::wait` |
| `nouveau_bo_new(cli, size, align, domain, tile_mode, tile_flags, sg, robj, bo)` | BO create | `NouveauBo::create` |
| `nouveau_bo_pin(bo, domain, contig)` / `_unpin(bo)` | pin/unpin | `NouveauBo::pin` |
| `nouveau_gem_ioctl_new` / `_pushbuf` / `_cpu_prep` / `_cpu_fini` / `_info` | legacy GEM ioctls (ABI16) | `Gem::*` |
| `nouveau_abi16_ioctl(...)` | classic ABI16 dispatch | `Abi16::ioctl` |
| `nouveau_uvmm_bind_ioctl` / `_uvmm_init_ioctl` / `_uvmm_create_ioctl` | modern uvmm ABI | `Uvmm::bind_ioctl` etc. |
| `nouveau_exec_ioctl_exec(...)` | modern EXEC ioctl | `Exec::ioctl` |
| `nouveau_sched_init(...)` / `_sched_fini(...)` | drm-sched bridging | `Sched::init` |
| `nouveau_svm_init(drm)` / `_fini(drm)` / `_svmm_init_ioctl(...)` | SVM init | `Svm::init` |
| `nouveau_dmem_init(drm)` / `_fini(drm)` / `_migrate_to_dmem(...)` | device memory | `Dmem::*` |
| `nouveau_fence_new(...)` / `_emit(...)` / `_wait(...)` / `_done(...)` | fence | `Fence::*` |
| `nouveau_pmops_*` | pm hooks | `Pmops::*` |
| `nouveau_acpi_init(drm)` / `_acpi_dsm(...)` | ACPI Optimus | `Acpi::*` |
| `nouveau_bios_run_init_table(...)` / `_bios_run_detect_table(...)` | VBIOS init scripts | `Vbios::run_init` |
| `nvkm_device_new_pci(...)` / `_del(...)` | NVKM device alloc | `NvkmDevice::new_pci` |
| `nvkm_device_init(...)` / `_fini(...)` | NVKM init | `NvkmDevice::init` |
| `nvkm_subdev_init(...)` | subdev init | `Subdev::init` |
| `nvkm_engine_init(...)` | engine init | `Engine::init` |
| `r535_gsp_rm_*` / `r570_gsp_rm_*` | GSP RM ops | `GspRm<R>::*` |
| `nvkm_gsp_msg_send(...)` / `_msg_recv(...)` | GSP RPC | `Gsp::msg_send` / `msg_recv` |
| `nvkm_gsp_client_new(...)` | GSP RM client | `GspClient::new` |

## Compatibility contract

REQ-1: PCI ID table — every NVIDIA GPU silicon from NV04 (PCI 0x10de:*) through Blackwell. Frozen.

REQ-2: UAPI — `include/uapi/drm/nouveau_drm.h`. Two ABI generations:
- **ABI16** (legacy): GEM_NEW, GEM_PUSHBUF, GEM_CPU_PREP/FINI, GEM_INFO, NOTIFIEROBJ_ALLOC, CHANNEL_ALLOC/FREE, GROBJ_ALLOC. Used pre-Turing or as a fallback.
- **Modern**: VM_INIT, VM_CREATE, VM_BIND, EXEC, GETPARAM, SETPARAM. Used Turing+ via GSP.

REQ-3: Two execution paths — Legacy (pre-Turing) and GSP (Turing+). Detected at probe; chip class determines which path is taken.

REQ-4: GSP firmware — Required for Turing+. NVIDIA-signed `nvidia/<chip>/gsp/{booter_load,booter_unload,bootloader,gsp}*` files. Without them, Turing+ chips bind nouveau but can't use 3D/compute.

REQ-5: NVKM subdev list — every chip class has per-class implementations for: PMC, PRIV, BUS, FB, INSTMEM, MMU, MC, FAULT, GPIO, I2C, MXM, CLK, THERM, PMU, FUSE, GSP, ACR (anti-CounteRfeiting), top, vfn (VF for SR-IOV future).

REQ-6: Engine list per chip — GR (graphics), CE (Copy Engine), DEC (video decode), MSVLD/MSENC/MSPDP/MSPP (legacy video), SEC (security), SW (software), DISP (display), FIFO (channel scheduler).

REQ-7: Channel model — per-engine channel, push buffer + indirect-buffer protocol, fence emit/wait.

REQ-8: TTM-backed BOs in: VRAM, GART, sysmem. Per-chip-class BO copy method (bo0039, bo5039, etc. for Fermi/Kepler/...).

REQ-9: UVMM (modern VM_BIND) — drm-gpuvm framework, ops: BIND, UNBIND, MAP, UNMAP, PREFETCH.

REQ-10: SVM — HMM-backed unified virtual memory. CPU mmaps a range; GPU faults on first access; pages migrated. Mmu-notifier coherence.

REQ-11: DMEM — device VRAM exposed as ZONE_DEVICE pages backing SVM. CPU can mmap; GPU can fault.

REQ-12: ACPI Optimus — hot-add/remove for laptops with iGPU+dGPU; runtime PM coordinates D0/D3hot/D3cold.

REQ-13: VBIOS init — running VBIOS init scripts at probe for legacy chips; GSP RM does its own VBIOS on modern chips.

REQ-14: vGPU / SR-IOV — limited; mostly host-side for future MIG-class workloads.

## Acceptance Criteria

- [ ] AC-1: GTX 1080 (Pascal, GP104) probes via legacy path; modesetting works; OpenGL via Mesa works; Vulkan via NVK is limited.
- [ ] AC-2: RTX 3080 (Ampere, GA102) probes via GSP path; GSP FW loads + RM client established; full Vulkan via NVK works; vkCTS pass rate matches upstream.
- [ ] AC-3: RTX 4090 (Ada, AD102) — GSP path; Vulkan + compute via NVK works.
- [ ] AC-4: RTX 5090 (Blackwell, GB202) — GSP path; Vulkan via NVK works (when GSP RM r570+ available).
- [ ] AC-5: Optimus laptop: iGPU+dGPU; nouveau-dGPU enters D3cold when idle; first-frame after wake within budget.
- [ ] AC-6: SVM smoke: CUDA-like unified-memory program (via NVK + extension) — CPU writes, GPU reads — works via HMM migration.
- [ ] AC-7: GSP RPC fuzz survival: malformed RPC response from a debugfs hook; driver rejects + recovers cleanly.
- [ ] AC-8: GPU recovery on hang: shader loop hangs an engine; FIFO TDR fires; channel killed; new channel works.
- [ ] AC-9: Suspend/resume: S3 + S4 cycles; modern silicon preserves GSP state.

## Architecture

**Two-path duality.** At probe, NVKM detects the chip class and decides:
- Pre-Turing (chip class < TU100): legacy direct register access. Driver programs FB, MMU, GR, CE, FIFO directly. ABI16 is the primary UAPI.
- Turing+ (TU100+): GSP firmware path. Driver loads GSP firmware (booter + bootloader + GSP_RM), establishes RPC client, delegates most GPU mgmt to GSP. Modern UAPI (uvmm/exec).

Rookery models as `NouveauDev<Legacy>` vs `NouveauDev<Gsp>`. The two types share trait methods where semantics match (BO create, fence) but diverge on engine/channel/VM ops.

**NVKM framework.** NVKM = Nouveau Kernel Module — the chip-specific HW abstraction layer underneath nouveau (the DRM-facing driver). Each subdev/engine has a per-chip-class implementation:
- **subdev**: PMC, PRIV, BUS, FB, MMU, INSTMEM, FAULT, GPIO, I2C, MXM, BIOS, CLK, THERM, PMU, FUSE, MC, GSP, ACR, top, vfn.
- **engine**: GR (graphics), CE (copy engines, 1-N), DEC/MSVLD/MSENC/MSPDP (video — generation specific), SEC (security), SW (software), DISP (display), FIFO (channel scheduler), NVDEC/NVENC/NVJPG (modern video).

Each .c file (e.g. `nvkm/subdev/mmu/ga100.c`) implements the per-class hooks. NVKM iterates the subdev list at init, calling each subdev's `.init` / `.fini` in proper ordering.

**GSP RM.** GSP (GPU System Processor) runs NVIDIA-signed firmware. The firmware is "RM" — Resource Manager — the same RM that NVIDIA's proprietary driver uses, but compiled for GSP. nouveau acts as an RPC client:
1. Load booter FW (signed, simple bootloader).
2. Booter loads bootloader.
3. Bootloader loads GSP RM.
4. Driver establishes RM client over GSP RPC.
5. All HW mgmt — channel create, VM ops, engine control, frequency, power — happens via RPC.

GSP RM versions evolve. Each major version (r535, r570, r575) has different message ABI. Driver tracks per-version via `nvkm/subdev/gsp/rm/<version>/`. A binary signed for one RM version won't run with another driver version.

**Channel + fence.** Each engine context is a "channel". Channels live in a runlist. Driver pushes work via push-buffer indirection. Each push includes a fence emit at end; driver waits or polls.

**UVMM (modern VM_BIND).** Uses the kernel's drm-gpuvm framework. VM_BIND is the only VM op (no relocs). Asynchronous via a bind queue; sync via drm-sched fences.

**SVM / HMM.** CPU + GPU share a virtual address space. Migration on demand: CPU faults on GPU-resident page → migrate to system RAM; GPU faults on CPU-resident page → migrate to VRAM (via DMEM). mmu-notifier keeps mappings coherent.

**Optimus runtime PM.** Laptop with iGPU+dGPU; dGPU is `nouveau`. When no apps use the dGPU, drop it to D3cold (full power off). On app start (xrandr setprovideroutputsource): ACPI _DSM wake; PCI re-init; dGPU re-trains. Lots of historical bugs here (CVE class around PM races).

## Hardening

- Each ioctl validated against the chip-class capability (legacy ioctls refused on GSP path; modern refused on legacy path where unsupported).
- GSP RPC response validated: length, msg-id, FW-signed envelope checked.
- NVKM subdev .init/.fini ordering enforced; partial-init rollback on failure.
- Channel push-buffer addresses validated against the channel's VM.
- VBIOS init script interpreter has bounded iteration + opcode allowlist.
- DMEM page migrations rate-limited.
- SVM mmu-notifier invalidates serialized with in-flight GPU work.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — every nouveau ioctl has a typed arg struct; bounded `copy_from_user`. Push-buffer data is user-supplied and lives in user-mapped BOs; never copied to/from kernel stack.
- **PAX_KERNEXEC** — `nouveau_drm_driver`, ioctl table, GEM ops, NVKM subdev/engine ops, GSP RPC dispatch, NVIF dispatcher, fence ops all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — per-syscall stack-rerandomize on every ioctl entry.
- **PAX_REFCOUNT** — `nouveau_drm` refcount, per-channel refcount, per-BO refcount, per-uvmm refcount, per-svmm refcount, GSP client refcount, NVIF object refcount all saturating.
- **PAX_MEMORY_SANITIZE** — BO backing cleared on TTM eviction. SVM/DMEM pages cleared on migration boundaries. GSP RPC buffers cleared after send. VBIOS init-script interpreter scratch cleared.
- **PAX_UDEREF** — SMAP/SMEP on every user-facing entry.
- **PAX_RAP / kCFI** — drm_driver ops, GEM ioctl dispatch, NVKM subdev/engine vtables, GSP RPC handler, NVIF dispatch, fence ops, ACPI hooks all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs (`/sys/kernel/debug/dri/N/nouveau_*`) scrubbed for non-root; reveals NVKM internals, GSP client pointers, channel/VM addresses.
- **GRKERNSEC_DMESG** — nouveau verbose logs (NVKM init trace, GSP RPC trace, channel state changes) restricted from non-root.
- **DRM_AUTH + render-node split** — KMS ops require master; render ops accept authenticated render-node clients.
- **GSP firmware signature mandatory** — booter, bootloader, GSP RM all NVIDIA-signed. PSP-equivalent FALCON secure-boot enforces signature; unsigned refused. LOCKDOWN_INTEGRITY_MAX disables FW updates.
- **GSP RPC opcode allowlist** — accept only the known message opcodes for the negotiated RM version; unknown opcodes drop with rate-limited warn (closes "compromised GSP forges new opcode" class).
- **GSP RPC response length validated per-opcode** — closes parser-OOB.
- **VBIOS init-script interpreter bounded** — opcode allowlist + iteration cap; closes a path where a malformed VBIOS could DoS or escape the interpreter.
- **Per-channel VMID isolation** — channel push-buffers access only the channel's VM; cross-channel access refused at MMU level (HW-enforced).
- **SVM mmu-notifier sync strict** — `MmuNotifier` registration held by typed `Svmm`; release-on-destroy ordering enforced (closes CVE-2024-26695 class — UAF on SVM teardown).
- **DMEM migration rate-limit** — per-process migration rate-limited; closes a "spammer triggers thrashing migrations" DoS class.
- **FIFO TDR per-channel + rate-limit** — channel hang triggers TDR; rate-limited 3 per 60s/channel; further triggers permanent channel offline (not whole-GPU).
- **ACPI _DSM allowlist** — only known _DSM functions (Optimus power, mux switching) accepted; closes parser-bug class.
- **GSP RM client per-process** — each process gets its own GSP RM client; closes cross-process RM-state leak.
- **NVIF object refcount strict** — NVIF object refcount underflow at remove is hard panic.
- **Compute queue priority CAP_SYS_NICE** — same as other GPU drivers.
- **NV-DEC/NV-ENC sessions gated by render-node** — video sessions require authenticated render node.
- **MIG (Multi-Instance GPU) partitioning gated by CAP_SYS_ADMIN** — on Ampere+ datacenter cards (A100/H100), MIG partition changes are CAP_SYS_ADMIN; closes a tenant-isolation bypass class.
- **PXP-equivalent (NVIDIA's protected content)** — when GSP RM advertises content-protect, encrypted output BOs un-mmap-able without keys.
- **Falcon authentication** — pre-GSP-era engines used FALCON-signed firmware; signature mandatory on every engine that requires it.

Per-doc rationale: nouveau is unique among the GPU drivers — two distinct execution paths (legacy direct + GSP-FW) covering 25+ years of silicon. Historical CVEs concentrate in SVM (mmu-notifier UAF), channel state machine, push-buffer parsing (pre-GSP), and DMEM migration races. The GSP path simplifies driver-side complexity but introduces GSP-RPC as a new attack surface — defended by opcode allowlist + response length validation + signed-FW boundary. The Rust translation's NouveauDev<Legacy|Gsp> typing closes the cross-path bug class (calling legacy op on GSP device = compile error). NVIF as typed API closes a major bug source in the NVKM↔nouveau interface. Per-RM-version GspRm<R> typing closes ABI-mismatch bugs across FW upgrades.

## Open Questions

- [ ] Q1: Legacy path maintenance — pre-Turing silicon represents a decreasing share of deployed hardware. Should Rookery deprecate Legacy path entirely (refuse pre-Turing) or maintain it? Recommendation: maintain — too much installed base; isolate behind `NouveauDev<Legacy>` typing so changes don't cross.
- [ ] Q2: GSP RM version coverage — r535, r570 currently; r575+ coming. Per-version Rust crate, lift dispatch to runtime selection at probe.
- [ ] Q3: NVK driver Vulkan completeness — depends on GSP path features. Track parity with NVIDIA proprietary driver.
- [ ] Q4: MIG (datacenter GPU partitioning) — supported by nouveau? Currently no. If Rookery wants MIG, requires GSP RM support that nouveau hasn't integrated yet.
- [ ] Q5: SR-IOV / vGPU — proprietary driver supports this; nouveau doesn't currently. Defer.
- [ ] Q6: Optimus PM races — long history of bugs. Rust translation must explicitly type the D0/D3hot/D3cold transitions across both the GPU and the ACPI _DSM state.

## Verification

- **Kani SAFETY**: prove NouveauDev<Legacy>::method cannot accidentally call into GSP-only code. Prove SVM mmu-notifier release ordering (closes CVE-2024-26695 class structurally).
- **TLA+**: model the GSP RPC message-id allocator + response routing; check no message-id collision / no UAF on slow response.
- **Verus**: functional spec of the GSP RPC framer — encode/decode roundtrip for every supported opcode produces identical bytes (no information loss).
- **Kani+Verus**: invariant that every channel has a valid VM at submit; every VM has all its mapped BOs tracked; no leaks at VM close.
- **Integration**: full Mesa NVK Vulkan compliance across Turing/Ampere/Ada/Blackwell test silicon; OpenGL Mesa NVC0 driver compliance for legacy chips; SVM workload (CUDA-style unified memory) on Ampere+; Optimus PM stress (1000 D0/D3cold cycles); GSP RM upgrade smoke (FW update via devlink).
- **Fuzz**: GSP RPC response fuzzer; push-buffer fuzzer for legacy path; VBIOS init-script fuzzer.
- **Penetration**: cross-process SVM access attempt — refused. Cross-channel push-buffer access attempt — HW-rejected.

## Out of Scope

- NVKM per-subdev/engine details (gr, ce, fifo, mmu, fb, instmem, ...) — separate Tier-3 family
- GSP RM version-specific differences — separate Tier-3 per major version (r535, r570, etc.)
- Display engine (DispNv50/DispC) — separate Tier-3
- NVK Vulkan userspace driver — out of kernel scope
- NVIDIA proprietary driver (`nvidia.ko`) — out of scope (not upstream)
- MIG / vGPU / SR-IOV — punted per Q4/Q5
- Falcon firmware internals — out of scope
