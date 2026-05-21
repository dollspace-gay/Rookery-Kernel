# Tier-3: drivers/gpu/drm/amd/amdgpu/{vcn_v4_0,vcn_v4_0_3,vcn_v4_0_5,amdgpu_vcn}.c — AMD VCN v4 video codec engine (AV1/H.265/H.264 decode + encode on RDNA3)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpu/drm/00-overview.md
depends-on:
  - drivers/gpu/drm/amd/amdgpu-core.md
upstream-paths:
  - drivers/gpu/drm/amd/amdgpu/vcn_v4_0.c                        (~2335 lines: VCN 4.0 — Navi 31/32/33 + Phoenix1)
  - drivers/gpu/drm/amd/amdgpu/vcn_v4_0.h
  - drivers/gpu/drm/amd/amdgpu/vcn_v4_0_3.c                      (~varies: MI300-family VCN variant)
  - drivers/gpu/drm/amd/amdgpu/vcn_v4_0_3.h
  - drivers/gpu/drm/amd/amdgpu/vcn_v4_0_5.c                      (Phoenix2/Hawk Point/Strix VCN variant)
  - drivers/gpu/drm/amd/amdgpu/vcn_v4_0_5.h
  - drivers/gpu/drm/amd/amdgpu/amdgpu_vcn.c                      (~1645 lines: cross-version VCN helpers — FW load, IB-test, RB-test, IDLE_WORK, ring funcs)
  - drivers/gpu/drm/amd/amdgpu/amdgpu_vcn.h
  - include/asic_reg/vcn/vcn_4_0_*_offset.h
  - include/asic_reg/vcn/vcn_4_0_*_sh_mask.h
-->

## Summary

VCN (Video Core Next) is AMD's dedicated video codec block — fixed-function silicon for hardware-accelerated decode + encode. VCN v4 is the version paired with RDNA3 silicon: Navi 31/32/33, Phoenix1/2, Hawk Point, Strix Point, plus MI300-series CDNA3 datacenter accelerators (vcn_v4_0_3 variant). For every Radeon-RX-using YouTube playback, Netflix stream, OBS streaming session, Zoom video call, Twitch broadcast, FFmpeg media pipeline, VCN is the engine doing the work — and on a modern CPU+GPU it's the difference between 5% GPU usage + cool laptop vs 100% CPU + fan ramp.

VCN v4 supports:
- **Decode**: H.264 (AVC) up to 4K@60, H.265 (HEVC) up to 8K@30, AV1 up to 8K@60 (new in VCN v4), VP9 up to 8K@30, MPEG-2/4. JPEG decode in dedicated sub-engine.
- **Encode**: H.264 + H.265 + AV1. Encode quality preset 0..7 (slowest=best).
- Multi-instance: 2 VCN engines on Navi 31, 1 on Navi 32/33/Phoenix, multi-instance on MI300.
- JPEG codec in dedicated `JPEG` sub-engine (separate microcode).

Architecturally, VCN is similar to GFX/SDMA in driver framework — an IpBlock with microcode loading, ring bring-up, IRQ handlers, power-gating — but the packet protocol is the OpenCL-derived "VCN session" protocol (very different from PM4 or SDMA opcodes). Userspace (Mesa va-api driver) sets up decode/encode sessions, allocates output framebuffers, then submits VCN session commands via amdgpu_cs. Kernel-side validates the commands minimally — most of the per-frame work is in the FW + silicon.

This Tier-3 covers ~4000 lines: vcn_v4_0.c + amdgpu_vcn.c. Per-revision variants (v4_0_3 for MI300, v4_0_5 for Strix) covered briefly here, in-depth in their own follow-up docs if material divergence.

## Rust translation posture

VCN is simpler than GFX but has unique surface: codec sessions, per-session FW state, codec-specific FW images. Translation:

- **`amdgpu-vcn-v4` crate** implements `IpBlock`. Self-contained microcode + ring + IRQ.
- **Multi-instance.** VCN has up to N engines per ASIC (2 on Navi 31, more on MI300). Typed `VcnInstance` per instance; each owns its rings.
- **Ring types.** Each VCN instance has multiple rings: decode ring, encode rings, JPEG ring. Typed `Ring<DecodeMode>`, `Ring<EncodeMode>`, `Ring<JpegMode>`.
- **Session-based codec state.** Userspace creates a codec "session" (typically per-stream) and submits per-frame commands. Driver keeps minimal per-session state; the FW owns most of it.
- **Codec-specific FW.** AV1 has its own FW image distinct from HEVC; loaded as needed. Per-codec FW signature checked.
- **Power-gating.** VCN is heavily power-gated when idle (saves substantial laptop battery); driver auto-PG after `pg_idle` ms.
- **Idle work.** Long-running task that polls instance idleness + initiates PG.

Grsec is mandatory: VCN takes user-supplied bitstream data (video frames). Malformed bitstreams have produced silicon-level bugs (HW codec faults that crash the engine). Driver-side validation is minimal; recovery on engine fault is the defense.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `vcn_v4_0_ip_funcs` | IpBlock for VCN 4 | `VcnV4IpBlock` |
| `vcn_v4_0_early_init` / `_sw_init` / `_hw_init` / `_hw_fini` / `_sw_fini` | init/fini | `VcnV4IpBlock::*` |
| `vcn_v4_0_start` / `_stop` | per-instance start/stop | `VcnInstance::start` / `stop` |
| `vcn_v4_0_dec_ring_emit_*` | decode ring packet emit | `Ring<Decode>::emit_*` |
| `vcn_v4_0_enc_ring_emit_*` | encode ring packet emit | `Ring<Encode>::emit_*` |
| `vcn_v4_0_jpeg_ring_emit_*` | JPEG ring packet emit | `Ring<Jpeg>::emit_*` |
| `vcn_v4_0_set_dec_ring_funcs` / `_enc_ring_funcs` / `_jpeg_ring_funcs` / `_irq_funcs` | install ops | `VcnRingOps` impls |
| `vcn_v4_0_pause_dpg_mode` / `_unpause` | DPG (deep power-gating) mode | `Dpg::pause` / `unpause` |
| `vcn_v4_0_enable_clock_gating` / `_disable_clock_gating` | CG | `Cg::enable` / `disable` |
| `vcn_v4_0_set_powergating_state` | PG state set | `Pg::set` |
| `vcn_v4_0_init_static_metadata` | per-instance static metadata | `StaticMeta::init` |
| `amdgpu_vcn_sw_init(adev)` / `_sw_fini(adev)` | cross-version sw init | `AmdgpuVcn::sw_init` / `_sw_fini` |
| `amdgpu_vcn_setup_ucode(adev)` | FW load coordination | `Ucode::setup` |
| `amdgpu_vcn_suspend(adev)` / `_resume(adev)` | suspend/resume | `AmdgpuVcn::suspend` / `_resume` |
| `amdgpu_vcn_idle_work_handler(work)` | PG-after-idle | `IdleWork::tick` |
| `amdgpu_vcn_ring_begin_use(ring)` / `_ring_end_use(ring)` | wake/idle for ring | `Ring::begin_use` / `_end_use` |
| `amdgpu_vcn_dec_ring_test_ring(ring)` / `_enc_ring_test_ring(...)` / `_jpeg_ring_test_ring(...)` | ring tests | `RingTest::run` |
| `amdgpu_vcn_dec_ring_test_ib(ring, timeout)` / `_enc_ring_test_ib(...)` / `_jpeg_ring_test_ib(...)` | IB tests | `IbTest::run` |
| `amdgpu_vcn_dec_msg_buffer_init(...)` | decode msg buffer init | `DecMsg::init` |
| `vcn_v4_0_unified_ring_emit_ib` | unified ring IB emit | `UnifiedRing::emit_ib` |
| `vcn_v4_0_process_interrupt(...)` | IRQ handler | `Irq::on_interrupt` |
| `vcn_v4_0_set_ras_funcs` | RAS hookup | `VcnRas::set_funcs` |

## Compatibility contract

REQ-1: ASIC support — VCN 4.0 (Navi 31/32/33 + Phoenix1), VCN 4.0.3 (MI300), VCN 4.0.5 (Phoenix2/Hawk Point/Strix). Per-revision microcode + register layouts.

REQ-2: Microcode files — `amdgpu/vcn_4_0_*.bin` per ASIC. PSP-signed. Frozen.

REQ-3: Codec capabilities — exposed via `AMDGPU_INFO_VIDEO_CAPS`: decode caps (H.264/H.265/AV1/VP9/MPEG-2/MPEG-4 with per-codec max resolution + max bitrate), encode caps (H.264/H.265/AV1 with quality presets, max resolution, max bitrate). Frozen against silicon capabilities.

REQ-4: Decode ring — single decode ring per instance; user submits via amdgpu_cs; ring per AMDGPU_HW_IP_VCN_DEC.

REQ-5: Encode rings — multiple encode rings per instance (typically 2); per AMDGPU_HW_IP_VCN_ENC. Higher count for higher concurrent stream count.

REQ-6: JPEG ring — separate AMDGPU_HW_IP_VCN_JPEG; for hardware JPEG decode/encode.

REQ-7: Unified ring (newer VCN 4) — combined decode + encode capability in a single ring for some workloads; codec selection per-IB.

REQ-8: Session state — userspace establishes a session via decode/encode session-start command. Kernel-side validates session count is within FW caps; per-session state held FW-side.

REQ-9: Power gating — when no rings active for `pg_idle` (typically 1s), VCN clock-gated + power-gated; transparent wake on next ring use.

REQ-10: DPG (Deep Power Gating) — additional power state where even the FW SRAM is gated off; restored on demand. Slower wake (msec) but lower idle power.

REQ-11: RAS (datacenter SKUs) — VCN UE counters integrated with amdgpu_ras.

REQ-12: Suspend/resume — full state preservation; codec sessions survive S3 (FW state saved + restored).

## Acceptance Criteria

- [ ] AC-1: RX 7900 XTX probes; VCN v4 enumerates 2 instances; microcode loads; decode/encode/JPEG rings ready; AMDGPU_INFO_VIDEO_CAPS reports expected codec caps.
- [ ] AC-2: VAAPI decode: `vainfo` lists VAProfileH264High, VAProfileHEVCMain, VAProfileAV1Profile0, etc.; `ffmpeg -hwaccel vaapi -i av1.mp4 -f null -` decodes at >2x realtime for 4K AV1 content.
- [ ] AC-3: VAAPI encode: `ffmpeg -i in.mp4 -c:v hevc_vaapi -b:v 5M out.mp4` encodes 1080p60 at >2x realtime.
- [ ] AC-4: Multi-stream: 4 concurrent 1080p H.265 decode streams on Navi 31; both VCN instances active per amdgpu_top; no frame drops.
- [ ] AC-5: PG cycle: idle the VCN for >pg_idle; verify PG entry via debugfs/dmesg; submit a frame; verify wake; latency < first-frame budget.
- [ ] AC-6: DPG cycle: deep power-gate via cmdline; first frame after DPG wake is timely (within ~50ms).
- [ ] AC-7: Suspend/resume: S3 with codec session active; resume; first post-resume frame decodes cleanly (FW state restored).
- [ ] AC-8: Hang recovery: submit a malformed bitstream that hangs the VCN; TDR fires; reset; subsequent decodes succeed.
- [ ] AC-9: JPEG: `vaapi_jpeg_decode` succeeds; multi-thread JPEG throughput at silicon spec.
- [ ] AC-10: AV1 specifically: `ffmpeg -c:v av1_vaapi -i 4k_av1.mp4` decodes at >60fps (validates new VCN v4 AV1 support).

## Architecture

**Per-instance init.** Each VCN instance bring-up:
1. Allocate microcode buffer; load via PSP (FW signed).
2. Init decode ring (1 per instance).
3. Init encode rings (typically 2 per instance).
4. Init JPEG ring (1 per instance).
5. Init static metadata (per-instance FW config).
6. Hook RAS.

**Ring funcs differ per ring type.** Decode ring uses one packet format; encode another; JPEG yet another. Each ring has its own `ring_funcs.emit_ib`, `emit_fence`, `emit_pipeline_sync`, etc. Rookery's typed Ring<Mode> ensures cross-mode emit is impossible.

**Session lifecycle.** Userspace (Mesa va-api):
1. Allocate session-context BOs.
2. Issue session-init IB.
3. Per frame, issue decode-frame or encode-frame IBs.
4. At session-end, issue session-uninit IB.

Driver-side: validate the IB targets the right ring, packet headers are well-formed at a high level (FW handles deep validation). Per-session state lives in FW SRAM; driver does not track session contents.

**Power gating.** Two granularities:
- Clock gating (CG): clock disabled when idle, instant wake. ~50% idle power saved.
- Power gating (PG): silicon powered off, ~50ms wake. ~90% idle power saved.
- Deep power gating (DPG): even FW SRAM powered off; FW re-loaded on wake. ~500ms wake. ~99% idle power saved.

Policy: aggressive PG when no rings have outstanding work; idle_work handler ticks every `pg_idle` (1s default) checking ring idleness.

**Recovery.** On VCN hang (decode/encode IB doesn't complete):
1. TDR per ring (using drm-sched).
2. VCN soft-reset.
3. Re-init the affected instance.
4. The hung ctx is marked guilty + killed.

**Multi-instance + MI300.** MI300 VCN 4.0.3 has many more instances (one per chiplet); driver handles instance enumeration via the IP discovery binary.

## Hardening

- Per-instance FW size validated before load.
- Ring buffer alloc bounded.
- Encode-ring count from FW caps, not user-supplied.
- Session count caps enforced FW-side; driver passes user IB through but checks for the session-init/-uninit pairing pattern.
- PG state machine has watchdog on missed PG entries.
- DPG re-load timeout enforced; past timeout, instance declared failed.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — no direct user surface; user IBs come through amdgpu_cs.
- **PAX_KERNEXEC** — IpBlock funcs, per-ring funcs, IRQ ops, RAS ops, idle-work handler all `const` after init in W^X memory.
- **PAX_RANDKSTACK** — covered at amdgpu_cs entry.
- **PAX_REFCOUNT** — per-instance refcount, per-ring refcount saturating; idle-work reschedule refcount saturating.
- **PAX_MEMORY_SANITIZE** — microcode load buffer cleared after PSP load. Session-context BOs zeroed at session-end (may contain DRM-protected video data). Encode output buffers freed after submit can carry encoded bitstream — sanitized at TTM eviction.
- **PAX_UDEREF** — N/A direct.
- **PAX_RAP / kCFI** — IpBlock funcs, per-ring funcs (decode/encode/jpeg/unified), IRQ dispatch, idle-work handler all kCFI-signed.
- **GRKERNSEC_HIDESYM** — debugfs VCN dumps (instance state, FW image addr, ring pointers) scrubbed for non-root.
- **GRKERNSEC_DMESG** — VCN verbose logs (codec session trace, frame-level events) restricted from non-root — can leak content-identification metadata.
- **Microcode signature mandatory** — every VCN/JPEG FW image PSP-signed; unsigned refused; LOCKDOWN_INTEGRITY_MAX disables FW updates.
- **Per-ring kCFI** — decode/encode/jpeg/unified rings each have distinct kCFI-signed emit functions; an attacker substituting a vtable cannot redirect emit to a wrong-mode handler.
- **DRM/HDCP content protection respected** — when a codec session has the HDCP-protect flag set, output BOs are marked encrypted; CPU cannot mmap them without holding HDCP key (enforced by PSP TA). Closes a path where DRM-protected playback content could be read by an unprivileged process.
- **TDR per ring with rate-limit** — VCN hang TDR rate-limited to 3 per 60s; further triggers declare the instance offline rather than infinite retry. Closes "malformed bitstream loops" DoS class.
- **Bitstream contents not parsed kernel-side** — only the IB packet headers + addresses are kernel-validated; the actual codec bitstream is FW + silicon parsed. Closes a class where complex bitstream parsers in the kernel are bug-prone.
- **DPG load signature re-checked** — when DPG reloads FW SRAM, the FW image is re-verified by PSP. Closes a path where a stale FW image (which could differ from the signed image) could be loaded on DPG wake.
- **Session-quota per-process** — driver tracks per-process decode/encode session count + enforces a per-process cap. Closes a path where one process exhausts all FW session slots, denying service to others.
- **VCN ring submit gated by render-node permission** — codec sessions are render-node ops; require an authenticated render node fd. Closes a path where an unprivileged process without DRM_AUTH could submit IBs.
- **JPEG output bounded** — JPEG decode output BOs have size validated against codec output dimensions; closes a path where forged dimensions could cause OOB write to a small output buffer.
- **PG-state-while-busy refused** — driver refuses to enter PG while any ring has outstanding work; the typed state machine encodes this. Closes a path where racy PG entry corrupts in-flight work.
- **Idle-work cancellation strict** — at module remove, idle-work cancelled before VCN teardown; no stale tick after instance destroyed (UAF prevention).

Per-doc rationale: VCN is the engine that processes attacker-controllable video bitstreams (every YouTube video, every screen share). Defense posture: kernel-side validation minimal (only IB-packet headers); FW + silicon do bitstream parsing. The reset+rate-limit machinery is the primary defense against malformed-bitstream-induced hangs. HDCP/DRM-protection respected when codec session uses encrypted output (protect by PSP TA). The Rust translation's typed Ring<Mode> closes the cross-mode-emit class. Per-instance + per-ring + per-session resource caps close DoS classes. Power-gating + DPG state machine typed to prevent racy PG-while-busy bugs.

## Open Questions

- [ ] Q1: AV1 encode tuning — AV1 encode quality preset matters a lot for streaming use cases. Should Rookery expose encode-preset as a devlink param, or only via amdgpu_cs IB attributes? Recommendation: IB attributes (already part of the codec ABI).
- [ ] Q2: DPG defaults — DPG saves substantial battery on laptops but wake latency hurts first-frame responsiveness. Recommendation: DPG default-off on plug-in, default-on on battery (sysfs hook into power_supply class).
- [ ] Q3: VCN unified-ring on Navi 31 — when to prefer unified over separate decode/encode? Mesa va-api will pick; driver mostly transparent.
- [ ] Q4: MI300 VCN 4.0.3 instance count — much higher than discrete GPUs. Verify scale tests cover MI300 scale specifically.
- [ ] Q5: HDCP content-protection — full HDCP enforcement requires display-pipeline coordination (DC). VCN-side support is half the picture; DC integration is the other half. Document the joint flow.

## Verification

- **Kani SAFETY**: prove typed Ring<DecodeMode> emit cannot reach an encode-ring submission. Prove PG entry cannot happen while any ring has outstanding work.
- **TLA+**: model PG/DPG/CG state machine with concurrent ring submission. Check no work-in-flight is lost across PG entries.
- **Verus**: functional spec of the codec-IB packet header parse — for valid packet, identifies the codec mode + session id; for malformed, returns -EINVAL.
- **Kani+Verus**: invariant that every active session has a corresponding session-init IB but no session-uninit IB; at instance destroy, all sessions are explicitly ended.
- **Integration**: full vaapi/ffmpeg codec matrix (H.264/H.265/AV1/VP9/MPEG decode and H.264/H.265/AV1 encode at 1080p/4K). 4-stream concurrent decode on Navi 31 sustained for 1h. PG cycle stress. DPG cycle stress. Suspend/resume across active session. Malformed bitstream hang recovery.
- **Fuzz**: codec-IB header fuzzer (mutate session-id, packet headers, addresses) — driver must reject or pass through to FW with no kernel oops.
- **Penetration**: unprivileged process tries to submit VCN IB without render-node auth — refused. HDCP-encrypted-output BO mmap from process without HDCP key — refused.

## Out of Scope

- Mesa va-api userspace driver — out of kernel scope
- Per-codec bitstream details — FW + silicon parsed
- VCN v1/v2/v3 (older AMD silicon — Vega 20, Navi 1x, Navi 2x) — separate Tier-3 per version
- VCN v5 (newer silicon, RDNA4) — future iteration
- AMF (AMD Media Framework, userspace closed-source equivalent of va-api) — not in scope
- Display-pipeline HDCP (DC side) — separate Tier-3 (amdgpu_dm)
