# Subsystem: sound/ — ALSA audio framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - sound/
  - sound/core/
  - sound/hda/
  - sound/pci/
  - sound/usb/
  - sound/soc/
  - sound/drivers/
  - include/sound/
  - include/uapi/sound/
-->

## Summary
Tier-2 overview for `sound/` — the Advanced Linux Sound Architecture (ALSA) subsystem. Owns the ALSA core PCM/MIDI/Sequencer/Control framework, HD-Audio codec dispatcher, per-bus driver families (PCI, USB, I2S/SoC, FireWire, ISA, AC'97), and the userspace ALSA ABI exposed via `/dev/snd/*`.

`sound/` is the lowest-priority Tier-2 subsystem for v0 because Rookery's primary v0 deployment context is server / containers / VMs where audio is rarely needed. **Per the strategic guidance for v0, sound/ stays as upstream-C-via-FFI.** This overview documents the FFI boundary and verifies the userspace ABI is preserved transparently.

## Upstream references in scope

| Category | Upstream paths | Planned design coverage |
|---|---|---|
| ALSA core | `sound/core/` (pcm.c, control.c, info.c, init.c, memory.c, oss/, seq/, …) | FFI'd in v0; Tier-3 doc documents the ABI surface preserved |
| HD-Audio | `sound/hda/`, `sound/pci/hda/`, `include/sound/hda*.h` | FFI'd in v0 |
| PCI sound drivers | `sound/pci/` | FFI'd in v0 |
| USB sound drivers | `sound/usb/` | FFI'd in v0 |
| ASoC (Audio for System-on-Chip) | `sound/soc/` | FFI'd in v0 (ARM-heavy; less relevant for x86_64 v0) |
| Firewire sound | `sound/firewire/` | OUT OF SCOPE for v0 (CONFIG=n) |
| ISA sound | `sound/isa/`, `sound/oss/` | OUT OF SCOPE for v0 (legacy) |
| ARM/MIPS/SPARC/PowerPC arch-specific sound | `sound/arm/`, `sound/mips/`, `sound/sparc/`, `sound/ppc/`, `sound/atmel/`, `sound/sh/`, `sound/parisc/` | OUT OF SCOPE for v0 (non-x86) |
| Misc helpers | `sound/i2c/`, `sound/spi/`, `sound/pcmcia/`, `sound/ac97/`, `sound/aoa/` | FFI'd or out-of-scope per relevance |
| AOA (Apple OnboardAudio) | `sound/aoa/` | OUT OF SCOPE (PowerPC Apple — non-x86) |

## Compatibility contract

Even though sound/ is FFI'd in v0, the userspace-visible ABI must be preserved (the C drivers run unchanged behind the FFI shim).

### `/dev/snd/*` surfaces

| Path | Owner doc | Compat level |
|---|---|---|
| `/dev/snd/controlC<n>` | `core.md` | ABI-identical |
| `/dev/snd/pcmC<n>D<n>{p,c}` | `core.md` | ABI-identical |
| `/dev/snd/seq` | `core.md` (sequencer subsystem) | ABI-identical |
| `/dev/snd/timer` | `core.md` | ABI-identical |
| `/dev/snd/midiC<n>D<n>` | `core.md` | ABI-identical |
| `/dev/snd/hwC<n>D<n>` | `core.md` | ABI-identical |

### `/proc/asound/*` and `/sys/class/sound/`

| Path | Owner doc | Compat level |
|---|---|---|
| `/proc/asound/{cards,devices,version,seq,...}` | `core.md` | Format-identical |
| `/sys/class/sound/card<n>/...` | `core.md` | Identical |

### ioctls

`SNDRV_PCM_IOCTL_*`, `SNDRV_CTL_IOCTL_*`, `SNDRV_TIMER_IOCTL_*`, `SNDRV_SEQ_IOCTL_*`, `SNDRV_HWDEP_IOCTL_*` — all preserved.

### Userspace tools

`alsa-utils` (`aplay`, `arecord`, `amixer`, `alsamixer`, `alsactl`, …), `alsa-lib`, `pulseaudio`, `pipewire` work unmodified.

## Requirements

- REQ-1: All ALSA `/dev/snd/*` device nodes appear with the same major:minor numbers and ioctl ABI as upstream.
- REQ-2: `/proc/asound/*` content is format-identical.
- REQ-3: HD-Audio probe + codec discovery on identical PCI hardware produces identical `dmesg` output and identical `/proc/asound/cards` entry.
- REQ-4: USB-Audio probe (USB-Audio class drivers) on identical hardware produces identical card enumeration.
- REQ-5: alsa-utils (aplay/arecord/amixer) operate unmodified against Rookery.
- REQ-6: PulseAudio + PipeWire operate unmodified.
- REQ-7: The FFI boundary between Rookery's Rust kernel and ALSA-C is well-defined: the FFI shim layer is documented in `core.md` and can be inspected via `pahole` to confirm struct-layout compat.
- REQ-8: When upstream's ALSA changes (between baselines), the FFI shim updates accordingly. Rookery does NOT pin to an old ALSA when rebaselining.

## Acceptance Criteria

- [ ] AC-1: `aplay -l`, `arecord -l`, `amixer scontents` output is identical to upstream on the same hardware. (covers REQ-1, REQ-3, REQ-5)
- [ ] AC-2: A WAV file plays through aplay and is captured through arecord identically (bit-identical sample stream after PCM conversion). (covers REQ-1, REQ-5)
- [ ] AC-3: `cat /proc/asound/{cards,devices,version}` is byte-identical. (covers REQ-2)
- [ ] AC-4: HD-Audio codec discovery on a HDA-equipped system matches upstream's enumeration. (covers REQ-3)
- [ ] AC-5: A USB-Audio class device (USB headset, microphone) enumerates correctly. (covers REQ-4)
- [ ] AC-6: PulseAudio's `pactl info` and `pactl list cards` output is identical. (covers REQ-6)
- [ ] AC-7: PipeWire's `pw-cli info` against the ALSA backend is identical. (covers REQ-6)
- [ ] AC-8: The FFI boundary is documented in `core.md` Tier 3; no Rookery code in `sound/` outside the FFI shim layer. (covers REQ-7)

## Architecture

### Layout map

```
.design/sound/
  00-overview.md       ← this document
  core.md              ← FFI shim doc — ABI preserved + how the shim works
```

(Single Tier-3 doc reflecting the FFI-only scope of sound/ in v0.)

### Cross-references

- `drivers/00-overview.md` — sound/pci/* and sound/usb/* drivers cross-reference the bus subsystems; even though FFI'd, the registration into PCI/USB driver model is exercised.
- `00-glossary.md` — none directly (sound terms are domain-specific and live in upstream ALSA docs).

### Rust module organization (informative)

- `kernel::sound::ffi` — thin FFI shim bringing the upstream C alsa-core into a Rookery `kernel::sound` namespace
- No further Rookery Rust code in v0

### Locking and concurrency

ALSA's internal locking is well-documented and preserved as-is (we don't touch it).

### Error handling

Standard kernel errnos pass through the FFI shim.

## Verification

### Layer 1: Kani SAFETY proofs
- FFI shim raw-pointer hand-off between Rookery Rust and ALSA C. Limited surface; harnesses focus on the boundary.

### Layer 2: TLA+ models
- (none — internal ALSA concurrency is upstream's; FFI shim has no novel concurrency)

### Layer 3: Kani harnesses
- FFI shim type-equivalence assertions

### Layer 4: opt-in
- (none for v0 — sound/ is FFI'd)

## Hardening

Placeholder per `00-overview.md` D6. Notable: ALSA has historically had a number of CVEs in its USB-Audio descriptor parser. v1+ reconsideration of porting USB-Audio driver to Rust would address this; for v0, FFI shim relies on upstream's continued maintenance.

## Resolved Decisions

### D1 (2026-05-09): sound/ Rust port deferred indefinitely
v0 keeps all of `sound/` as FFI'd C. No Rust port planned for v1+ either. Rookery's primary deployment context (server / containers / VMs) doesn't need audio in Rust. v1+ may revisit only if a CVE wave makes USB-Audio porting valuable; otherwise sound/ stays C.

## Open Questions

(none — all open questions for this subsystem document are resolved above)

## Out of Scope

- Full Rust port of any sound/ component — per Q1.
- Non-x86 sound drivers (sound/{arm,mips,sparc,ppc,atmel,sh,parisc,aoa}) — out of v0.
- Legacy ISA sound + OSS — out of v0 (CONFIG=n).
- Firewire sound — out of v0 (CONFIG=n).
- Implementation code.
