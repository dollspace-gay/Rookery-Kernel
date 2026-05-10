# Subsystem: init/ — kernel initialization

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - init/
  - usr/
  - include/linux/init.h
-->

## Summary
Tier-2 overview for `init/` — kernel initialization. Owns `start_kernel` (the first C function called once early-arch setup completes), the initramfs unpacker, the `init_task` (PID 0 / swapper) declaration, the rootfs mount logic (`do_mounts*`), the kernel command-line parsing dispatch, the kernel version/configs exposure, and the BogoMIPS calibration. Small subsystem (~13 files) but compat-critical: `start_kernel`'s ordering of subsystem initialization is observable via `dmesg` and indirectly via boot timing.

The arch tier (`arch/x86/00-overview.md` § kernel-platform.md) handles everything *before* `start_kernel`; this subsystem starts where arch hands over.

## Upstream references in scope

| Category | Upstream paths | Planned Tier-3 doc |
|---|---|---|
| `start_kernel` orchestration | `init/main.c` | `start-kernel.md` |
| init_task (PID 0 / swapper) | `init/init_task.c` | folded into `start-kernel.md` |
| Initramfs unpacking | `init/initramfs.c`, `init/initramfs_internal.h`, `init/noinitramfs.c`, `usr/` | `initramfs.md` |
| Rootfs mount + initrd compat | `init/do_mounts.c`, `init/do_mounts.h`, `init/do_mounts_initrd.c`, `init/do_mounts_rd.c` | `rootfs-mount.md` |
| BogoMIPS calibration | `init/calibrate.c` | folded into `start-kernel.md` |
| Kernel version metadata | `init/version.c`, `init/version-timestamp.c` | folded into `start-kernel.md` |

## Compatibility contract

### Boot output

- `dmesg` early-boot messages preserve their format and ordering (subsystem init order observable via grep).
- `Linux version <ver> (<builder>) (<gcc-or-clang>) #<n> SMP <date>` first-line format byte-identical.

### `/proc` surfaces

| Path | Owner doc | Compat level |
|---|---|---|
| `/proc/cmdline` (kernel boot command line) | `start-kernel.md` | Identical (echo of bootloader hand-off) |
| `/proc/version` | `start-kernel.md` | Format-identical |
| `/proc/version_signature` (Ubuntu, Debian use this) | `start-kernel.md` | Identical |
| `/proc/config.gz` (when CONFIG_IKCONFIG_PROC=y) | `start-kernel.md` (cross-ref `kernel/sysctl-ksysfs.md`) | Format-identical |

### Kernel command-line parameters

Every parameter documented in `Documentation/admin-guide/kernel-parameters.txt` is parsed identically. This is a substantial compat surface — hundreds of parameters across all subsystems. Each subsystem doc owns parsing for its own parameters; `init/00-overview.md` owns the dispatch.

### initramfs format

cpio newc / cpio crc / gzip / bzip2 / xz / lzma / lzop / lz4 / zstd compressed archives are all consumed identically. `usr/` is the build-helper that bakes a default initramfs into the kernel image.

## Requirements

- REQ-1: `start_kernel` orchestrates subsystem initialization in the same order as upstream — measurable via `dmesg` ordering when CONFIG_PRINTK_TIME=y.
- REQ-2: `init_task` declaration produces a task with the same `task_struct` fields as upstream's `init_task`.
- REQ-3: Initramfs unpacking accepts every encoding upstream accepts (cpio, plus the compressors above) and produces the same filesystem state.
- REQ-4: Rootfs mount logic (mounting initramfs as `/`, then handing off to a real root via `pivot_root` from the initramfs init script) preserves the upstream sequence.
- REQ-5: Initrd compat (legacy initrd-as-block-device, distinct from initramfs-as-cpio) is supported identically when `CONFIG_BLK_DEV_INITRD=y`.
- REQ-6: Kernel command-line parsing dispatches to the same per-parameter handlers as upstream.
- REQ-7: `/proc/cmdline`, `/proc/version`, `/proc/version_signature`, `/proc/config.gz` content is byte-identical.
- REQ-8: BogoMIPS calibration produces a value within ±5% of upstream's calibration on the same hardware (BogoMIPS is informational; tools observe it but tolerate noise).

## Acceptance Criteria

- [ ] AC-1: `dmesg | grep -E "(initializing|registered|setup|init|started|registered)"` ordering on Rookery and upstream produces the same sequence of events when sorted topologically (allowing for re-ordering of independent late_initcalls). (covers REQ-1)
- [ ] AC-2: `pahole init_task` produces an output structurally equivalent (same field names + counts + cache-line-zero offsets) on Rookery and upstream. (covers REQ-2)
- [ ] AC-3: A curated initramfs (uncompressed + xz + zstd + gzip variants) unpacks to byte-identical filesystem trees on Rookery and upstream. (covers REQ-3)
- [ ] AC-4: A standard distro boot path (initramfs → systemd-fstab → real root via pivot_root) succeeds on Rookery. (covers REQ-4)
- [ ] AC-5: Booting Rookery with a legacy `initrd=` parameter on a CONFIG_BLK_DEV_INITRD=y build succeeds. (covers REQ-5)
- [ ] AC-6: A kernel-cmdline test exercising every parameter documented in `Documentation/admin-guide/kernel-parameters.txt` produces byte-identical kernel state on Rookery vs. upstream (sysctl values, /sys files). (covers REQ-6)
- [ ] AC-7: `cat /proc/{cmdline,version,version_signature}` and `zcat /proc/config.gz` produce identical content. (covers REQ-7)
- [ ] AC-8: BogoMIPS reported by Rookery is within 5% of upstream on the same hardware. (covers REQ-8)

## Architecture

### Layout map

```
.design/init/
  00-overview.md       ← this document
  start-kernel.md      ← start_kernel orchestration + init_task + version + cmdline parsing dispatch
  initramfs.md         ← initramfs unpacker + cpio + decompressor selection
  rootfs-mount.md      ← do_mounts + initrd legacy + pivot_root semantics
```

### Cross-references

- `arch/x86/00-overview.md` § boot.md, kernel-platform.md — control reaches `start_kernel` via `arch/x86/kernel/head_64.S` then `head64.c::x86_64_start_kernel`.
- `kernel/00-overview.md` — every late_initcall / arch_initcall / pure_initcall belongs to its target subsystem; init/ owns the dispatch.
- Every subsystem doc — each subsystem registers its own `Documentation/admin-guide/kernel-parameters.txt` entries and consumes them through `init/`'s dispatch.
- `00-glossary.md` — `start_kernel`, `boot_params`, `initramfs`, `bzImage`/`vmlinuz`/`vmlinux`.

### Rust module organization (informative)

- `kernel::init::main` — start_kernel
- `kernel::init::initramfs` — unpacker
- `kernel::init::cmdline` — parser dispatch
- `kernel::init::rootfs` — mount sequence

### Locking and concurrency

`start_kernel` runs single-CPU-only until `smp_init()` brings APs online; locking is a non-issue for most of init/. After SMP, any late initcall must use the standard kernel locking primitives.

### Error handling

`init/` errors are typically panics — `start_kernel` BUG-panics on unrecoverable init failure (memory init failed, no CPU bring-up possible, etc.). Initramfs-unpack errors are logged but tolerated (kernel boots without rootfs and panics later if no real root is mountable).

## Verification

### Layer 1: Kani SAFETY proofs
- Initramfs cpio header parsing
- Decompressor invocation across format variants

### Layer 2: TLA+ models
- (none mandatory — start_kernel is single-threaded; no novel concurrency)

### Layer 3: Kani harnesses
- cpio header struct validation harness

### Layer 4: opt-in
- cpio parser correctness via Creusot — tractable, valuable for input-format security.

## Hardening

Placeholder per `00-overview.md` D6.

## Open Questions

(none for v0 init/ scope — well-defined boot-orchestration surface; per-Tier-3 docs may surface ambiguities)

## Out of Scope

- arch-specific pre-`start_kernel` code (owned by arch/x86/00-overview.md)
- Bootloader (GRUB, systemd-boot) — accepted as upstream contract
- Implementation code — `.design/` contains specs only
