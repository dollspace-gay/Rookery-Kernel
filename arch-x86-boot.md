---
title: "Tier-3: arch/x86/boot — x86_64 boot path"
tags: ["design-doc", "tier-3", "arch-x86"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the x86_64 boot path: from bootloader hand-off through real-mode setup, protected-mode transition, decompression, paging activation, long-mode entry, and arrival at `start_kernel`. Owns the byte-identical implementation of the **x86 boot protocol** (bzImage header + boot_params), the **EFI hand-off** path, the **SMP AP-bringup trampoline** (real-mode + long-mode portions), the **SEV-SNP and TDX guest startup** paths, the **kexec hand-off** (REQ-15 of `00-overview.md`), and the **purgatory** stub running between kexec'd kernels.

This is the most-architecture-bound Tier-3 doc in the corpus. The boot protocol is fixed by an external contract (the boot loader). REQ-1 through REQ-15 of `00-overview.md` mostly land here.

### Requirements

- REQ-1: bzImage `setup_header` is byte-identical: every field at the same offset with the same semantics. A Rookery `bzImage` boots under stock GRUB / systemd-boot / kexec / qemu unmodified.
- REQ-2: `boot_params` zero page is byte-identical: every field at the same offset with the same semantics. Bootloaders fill it identically; Rookery reads it identically.
- REQ-3: EFI hand-off ABI is byte-identical (`handover_offset`, ImageHandle/SystemTable/boot_params register convention; mixed-mode thunk).
- REQ-4: Decompression supports every compressor upstream supports (gzip, bzip2, lz4, lzma, lzo, xz, zstd) per `CONFIG_KERNEL_*` selection. Decompressed output is byte-identical.
- REQ-5: KASLR (`arch/x86/boot/compressed/kaslr.c`) randomizes the kernel base + physical-load address per `kaslr_get_random_long` semantics; default-on (CONFIG_RANDOMIZE_BASE=y per `00-security-principles.md` knob inventory).
- REQ-6: AMD SEV-SNP guest startup (#VC handlers, GHCB protocol) preserves the architectural contract — a Rookery kernel boots correctly inside a SEV-SNP confidential VM.
- REQ-7: Intel TDX guest startup (TDCALL/TDVMCALL) preserves the architectural contract — a Rookery kernel boots correctly inside a TDX trust domain.
- REQ-8: kexec hand-off produces a destination-kernel state byte-identical to upstream's kexec — REQ-15 of `00-overview.md`. A Rookery kernel can kexec into upstream and vice versa.
- REQ-9: Purgatory verifies SHA-256 of the destination kernel image identically to upstream; an upstream-built purgatory and a Rookery-built purgatory produce identical hashes for identical inputs.
- REQ-10: AP bringup trampoline preserves its protocol with `secondary_startup_64`. The BSP-side handoff (filling `trampoline_header`, issuing INIT-SIPI-SIPI) is byte-identical.
- REQ-11: 5-level paging (`la57toggle.S`) toggles correctly on supporting CPUs per CR4.LA57; CR4.LA57 setting matches upstream's `CONFIG_X86_5LEVEL` selection.
- REQ-12: SBAT data (Secure Boot Advanced Targeting; `sbat.S`) is embedded with a Rookery-specific SBAT entry but preserves the upstream Linux SBAT entry too (so existing Secure Boot dbx revocations apply).
- REQ-13: Early `#VC` and `#PF` IDT (`idt_64.c` + `idt_handlers_64.S`) is correctly installed BEFORE decompression touches encrypted memory; otherwise SEV-SNP boot crashes.
- REQ-14: `start_kernel` is reached with the same architectural preconditions upstream guarantees: paging is on, IDT is set, GDT is set, BSP has its kernel stack, boot_params is mapped in the kernel direct-map, the e820 table is consumed, ACPI RSDP is recorded, EFI runtime services pointer is recorded.
- REQ-15: Hardening section per `00-security-principles.md` template. KASLR + early-boot KASAN init + paging-W^X-from-the-start are owned here.

### Acceptance Criteria

- [ ] AC-1: `qemu-system-x86_64 -kernel rookery-bzImage -initrd test-initramfs.cpio.gz` boots to the test userspace's PID 1 successfully. (covers REQ-1, REQ-2, REQ-4, REQ-5, REQ-14)
- [ ] AC-2: A bare-metal boot via GRUB on a UEFI x86_64 system boots to userspace identically to upstream. (covers REQ-1, REQ-2, REQ-3, REQ-14)
- [ ] AC-3: A `pahole` of `struct setup_header` and `struct boot_params` produces byte-identical layout (field names, offsets, sizes) on Rookery vs. upstream. (covers REQ-1, REQ-2)
- [ ] AC-4: Each compressor (gzip, bzip2, lz4, lzma, lzo, xz, zstd) is exercised by building 7 separate bzImage variants; each boots successfully under qemu. (covers REQ-4)
- [ ] AC-5: A KASLR-enabled boot logs a non-deterministic kernel base address (visible via `cat /proc/kallsyms | head` or `dmesg` early lines). Multiple boots produce different bases. (covers REQ-5)
- [ ] AC-6: A SEV-SNP-enabled qemu-kvm guest boots Rookery successfully (requires SEV-SNP-capable host hardware OR qemu's TCG-mode SEV emulation). (covers REQ-6, REQ-13)
- [ ] AC-7: A TDX-enabled qemu-kvm guest boots Rookery successfully (TDX-capable host required). (covers REQ-7)
- [ ] AC-8: `kexec_load` + `kexec_file_load` syscalls succeed on Rookery; running `kexec -e` jumps into a freshly-loaded Rookery image and a freshly-loaded upstream image. The mirror direction (upstream loads + jumps into Rookery) also succeeds. (covers REQ-8)
- [ ] AC-9: Purgatory's SHA-256 hash of a curated test image is byte-identical to upstream's purgatory's hash on the same input. (covers REQ-9)
- [ ] AC-10: An SMP boot on a multi-CPU system brings every AP online. `cat /proc/cpuinfo` lists every expected CPU. The trampoline data structures (visible via `vmlinux` symbol search) are byte-identical to upstream's. (covers REQ-10)
- [ ] AC-11: 5-level paging boots successfully on a CR4.LA57-capable CPU (qemu's `-cpu max,la57=on`). 4-level paging boots successfully on `-cpu max,la57=off`. (covers REQ-11)
- [ ] AC-12: A Secure Boot test verifies that Rookery's bzImage has a valid SBAT section and is rejected when Linux SBAT entry is revoked at the appropriate generation. (covers REQ-12)
- [ ] AC-13: A SEV-SNP boot test that injects a #VC during pre-decompression GHCB initialization completes successfully (verifies the early IDT is installed). (covers REQ-13)

### Architecture

### Stage breakdown

```
[Bootloader]   → bzImage in memory
   ↓ jumps to setup_header.code32_start (or EFI handover_offset)
[Real-mode setup]  arch/x86/boot/   (16-bit, ~30 .c/.S files)
   ↓ pmjump to protected mode
[Decompression]  arch/x86/boot/compressed/   (32-bit + 64-bit, decompresses payload)
   ↓ jumps to extracted kernel's entry
[Long-mode kernel head]  arch/x86/kernel/head_64.S
   ↓ calls x86_64_start_kernel
[Early C startup]  arch/x86/kernel/head64.c
   ↓ falls through to start_kernel
[start_kernel]  init/main.c   (arch-agnostic; covered in init/00-overview.md § start-kernel.md)
```

For SMP APs, the trampoline path is:

```
BSP issues INIT-SIPI-SIPI  →  AP starts in real mode at trampoline (arch/x86/realmode/rm/)
   ↓ trampoline transitions: real → protected → long mode
[secondary_startup_64]  arch/x86/kernel/head_64.S (separate entry from BSP path)
   ↓ AP joins running kernel
```

For kexec, the path is:

```
[Currently-running kernel]
   ↓ kexec_load / kexec_file_load syscall populates segment list
[machine_kexec_64.c::machine_kexec]
   ↓ relocate_kernel_64.S copies segments to target addresses
[Purgatory]  arch/x86/purgatory/
   ↓ verifies SHA-256, jumps to destination
[New kernel's entry]
```

### Rust module organization

- `kernel::arch::x86::boot::header` — bzImage `setup_header` struct definitions; `#[repr(C)]` layout matching upstream byte-for-byte
- `kernel::arch::x86::boot::params` — `boot_params` struct definitions; `#[repr(C)]`
- `kernel::arch::x86::boot::realmode` — real-mode setup wrapper (most of `arch/x86/boot/`'s .c files become functions in this module)
- `kernel::arch::x86::boot::compressed` — decompression dispatcher + KASLR + early identity paging + early IDT + SEV/TDX guest startup
- `kernel::arch::x86::boot::startup` — `arch/x86/boot/startup/` analog; bridges decompression to the long-mode kernel
- `kernel::arch::x86::boot::head64` — `head_64.S` (assembly) plus `head64.c::x86_64_start_kernel`
- `kernel::arch::x86::boot::trampoline` — `arch/x86/realmode/` analog
- `kernel::arch::x86::kexec` — `arch/x86/kernel/machine_kexec_64.c` analog
- `kernel::arch::x86::purgatory` — separate small crate (no_std, no alloc, no kernel — purgatory is its own freestanding binary)

Assembly portions (`head_64.S`, `pmjump.S`, `trampoline_64.S`, `relocate_kernel_64.S`, `entry64.S`, `sev-handle-vc.S`, `tdcall.S`, `efi-mixed.S`, `la57toggle.S`, `idt_handlers_64.S`) remain assembly. They live in `arch/x86/<dir>/<file>.S` per upstream convention; built via kbuild as upstream.

### Key data structures (Rust mirrors)

- `setup_header` — 200+ byte struct; `#[repr(C, packed)]`; exact byte-for-byte upstream layout
- `boot_params` — 4096-byte struct; `#[repr(C)]`
- `e820_table` / `e820_entry` — memory map representation
- `setup_data` linked list — extension data passed by bootloader (EFI memory map, DTB, RNG seed, etc.)
- `efi_info` — EFI runtime services pointer + memory map handle

### Locking and concurrency

The boot path is single-CPU (BSP-only) until `smp_init`. AP startup uses INIT-SIPI-SIPI which is hardware-serialized; the trampoline data structures are written by the BSP before issuing IPIs, so no kernel-side lock is required.

kexec runs with all interrupts disabled and (mostly) all CPUs stopped via `stop_machine`. No locks required during the actual relocation.

### Error handling

Boot-path errors are typically panics: a corrupt boot_params, a missing required CPU feature, a decompression failure are all unrecoverable. The Rust counterpart uses `kernel::panic!` per `00-rust-conventions.md` REQ-5 (NOT bare `core::panic!`).

EFI hand-off errors before `ExitBootServices` use the EFI status-return convention (`EFI_LOAD_ERROR`, etc.) so the firmware can recover gracefully.

### Out of Scope

- 32-bit kernel boot (`X86_32=y`) — out of scope per `arch/x86/00-overview.md` D1
- math-emu (`arch/x86/math-emu/`) — out of scope per `arch/x86/00-overview.md` D4
- VGA video legacy (`arch/x86/video/`) — modern systems boot via EFI framebuffer; covered in `drivers/00-overview.md`
- Bootloader code itself (GRUB, systemd-boot) — accepted as upstream contract
- Implementation code — `.design/` contains specs only

### upstream references in scope

| Stage | Upstream paths | Notes |
|---|---|---|
| **Bootloader hand-off + bzImage header** | `arch/x86/boot/header.S`, `arch/x86/boot/setup.ld`, `Documentation/arch/x86/boot.rst` | The bzImage's `setup_header` struct + magic numbers; what GRUB / systemd-boot / kexec-tools writes to |
| **boot_params zero page** | `arch/x86/include/uapi/asm/bootparam.h`, `arch/x86/include/uapi/asm/setup_data.h`, `Documentation/arch/x86/zero-page.rst` | The struct the bootloader fills with memory map, command line, framebuffer info, EFI info, ramdisk, etc. |
| **Real-mode setup (16-bit)** | `arch/x86/boot/header.S`, `arch/x86/boot/main.c`, `arch/x86/boot/cmdline.c`, `arch/x86/boot/memory.c`, `arch/x86/boot/cpu.c`, `arch/x86/boot/cpucheck.c`, `arch/x86/boot/cpuflags.c`, `arch/x86/boot/edd.c`, `arch/x86/boot/a20.c`, `arch/x86/boot/apm.c`, `arch/x86/boot/early_serial_console.c`, `arch/x86/boot/printf.c`, `arch/x86/boot/string.c`, `arch/x86/boot/tty.c`, `arch/x86/boot/version.c`, `arch/x86/boot/video.c`, `arch/x86/boot/video-bios.c`, `arch/x86/boot/regs.c`, `arch/x86/boot/copy.S`, `arch/x86/boot/bioscall.S`, `arch/x86/boot/pm.c`, `arch/x86/boot/pmjump.S` | Real-mode probing of CPU, memory map, video, ACPI tables; transition to protected mode |
| **Decompression (compressed kernel image)** | `arch/x86/boot/compressed/` — `head_64.S` (entry from real-mode/PE), `misc.c` (decompressor dispatch), `kaslr.c` (KASLR), `ident_map_64.c` (early identity-map page tables), `idt_64.c` + `idt_handlers_64.S` (early IDT for #VC and #PF), `mem.c`, `mem_encrypt.S` (AMD SME), `pgtable_64.c`, `efi.c` (EFI hand-off path), `acpi.c`, `cmdline.c`, `error.c`, `cpuflags.c`, `early_serial_console.c`, `kernel_info.S`, `mkpiggy.c`, `sev.c` + `sev-handle-vc.c` + `sev.h` (SEV-SNP guest), `tdx.c` + `tdx.h` + `tdx-shared.c` + `tdcall.S` (Intel TDX guest), `sbat.S`, `string.c` | Decompresses the compressed kernel image; sets up early identity paging; jumps to long-mode kernel |
| **Startup helpers (post-decompression, pre-`start_kernel`)** | `arch/x86/boot/startup/` — `efi-mixed.S` (EFI 32-on-64 mixed-mode hand-off), `gdt_idt.c`, `la57toggle.S` (5-level paging activation), `map_kernel.c`, `sev-shared.c`, `sev-startup.c`, `sme.c`, `exports.h`, `Makefile` | Cross-stage helpers between decompression and the long-mode kernel proper |
| **Long-mode kernel entry** | `arch/x86/kernel/head_64.S`, `arch/x86/kernel/head64.c` (`x86_64_start_kernel`), `arch/x86/kernel/setup.c` (`setup_arch`) | First C function (`x86_64_start_kernel`) called from long-mode head_64.S; runs early arch setup before `start_kernel` |
| **AP bringup (SMP startup)** | `arch/x86/realmode/init.c`, `arch/x86/realmode/rm/` — `header.S`, `trampoline_64.S`, `trampoline_32.S`, `trampoline_common.S`, `wakeup_asm.S`, `wakemain.c`, `reboot.S`, `stack.S`, `regs.c`, `bioscall.S`, `copy.S`, `realmode.lds.S` | Real-mode trampoline used to bring up secondary CPUs from BSP-issued INIT/SIPI, plus S3 wakeup |
| **kexec hand-off** | `arch/x86/kernel/machine_kexec_64.c`, `arch/x86/kernel/relocate_kernel_64.S` | The kexec_load + kexec_file_load syscalls' x86 backends; relocation logic that copies segments and jumps to the new kernel |
| **Purgatory** | `arch/x86/purgatory/` — `entry64.S`, `setup-x86_64.S`, `stack.S`, `purgatory.c`, `kexec-purgatory.S` | Tiny stub running between kexec'd kernels (verifies SHA-256 of next kernel image; jumps to its entry point) |

### compatibility contract

The boot path is the most contractually-rigid Tier-3 in the project. Every byte of the boot interface is dictated by the bootloader / firmware contract.

### bzImage header (real-mode setup section)

`arch/x86/boot/header.S` defines the `setup_header` struct at offset 0x1F1 in the kernel image. Field-by-field byte-identity required:

| Field | Offset | Notes |
|---|---|---|
| `setup_sects` | 0x1F1 | Number of 512-byte sectors of setup code |
| `root_flags` | 0x1F2 | Legacy mount flags |
| `syssize` | 0x1F4 | Protected-mode payload size |
| `ram_size` | 0x1F8 | Legacy ramdisk size |
| `vid_mode` | 0x1FA | Video mode hint |
| `root_dev` | 0x1FC | Default root device |
| `boot_flag` | 0x1FE | 0xAA55 magic |
| `jump` | 0x200 | x86 jump instruction |
| `header` | 0x202 | "HdrS" magic = 0x53726448 |
| `version` | 0x206 | Boot protocol version |
| `realmode_swtch` | 0x208 | Realmode-switch routine pointer |
| `start_sys_seg` | 0x20C | Legacy start-system-segment |
| `kernel_version` | 0x20E | Kernel version string offset |
| `type_of_loader` | 0x210 | Loader type |
| `loadflags` | 0x211 | Load flags (LOADED_HIGH, KASLR_FLAG, QUIET_FLAG, KEEP_SEGMENTS, CAN_USE_HEAP) |
| `setup_move_size` | 0x212 | Setup move size |
| `code32_start` | 0x214 | 32-bit kernel entry point |
| `ramdisk_image` | 0x218 | Ramdisk physical address |
| `ramdisk_size` | 0x21C | Ramdisk size |
| `bootsect_kludge` | 0x220 | Reserved |
| `heap_end_ptr` | 0x222 | Heap end pointer |
| `ext_loader_ver` | 0x224 | Extended loader version |
| `ext_loader_type` | 0x225 | Extended loader type |
| `cmd_line_ptr` | 0x228 | Command-line pointer |
| `initrd_addr_max` | 0x22C | Maximum initrd address |
| `kernel_alignment` | 0x230 | Kernel image alignment |
| `relocatable_kernel` | 0x234 | Relocatable flag |
| `min_alignment` | 0x235 | Minimum alignment exponent |
| `xloadflags` | 0x236 | Extended loadflags (CAN_BE_LOADED_ABOVE_4G, CAN_USE_VCRT, EFI_KEXEC, 5LEVEL, 5LEVEL_ENABLED) |
| `cmdline_size` | 0x238 | Cmdline maximum size |
| `hardware_subarch` | 0x23C | Hardware subarchitecture |
| `hardware_subarch_data` | 0x240 | Subarch data |
| `payload_offset` | 0x248 | Payload offset |
| `payload_length` | 0x24C | Payload length |
| `setup_data` | 0x250 | Linked list of setup_data structs |
| `pref_address` | 0x258 | Preferred load address |
| `init_size` | 0x260 | Init size |
| `handover_offset` | 0x264 | EFI hand-off entry offset |
| `kernel_info_offset` | 0x268 | kernel_info struct offset |

### `boot_params` (zero page)

The struct in `arch/x86/include/uapi/asm/bootparam.h` is the FULL hand-off vehicle. Byte-identical layout required:

- `screen_info` (legacy VGA/framebuffer info)
- `apm_bios_info` (legacy APM)
- `tboot_addr` (TBOOT verified-launch)
- `ist_info` (Intel SpeedStep info)
- `acpi_rsdp_addr` (ACPI RSDP address from bootloader)
- `hd0_info`, `hd1_info` (legacy)
- `sys_desc_table` (legacy)
- `olpc_ofw_header` (OLPC)
- `ext_ramdisk_image`, `ext_ramdisk_size`, `ext_cmd_line_ptr` (extended pointers for >4GB)
- `cc_blob_address` (Confidential Computing blob)
- `edid_info`
- `efi_info` (EFI memory map handle, system table)
- `alt_mem_k`, `scratch`, `e820_entries`, `eddbuf_entries`, `edd_mbr_sig_buf_entries`, `kbd_status`, `secure_boot`
- `e820_table[E820_MAX_ENTRIES_ZEROPAGE]` (legacy e820 entries)
- `eddbuf[EDDMAXNR]`
- `edd_mbr_sig_buffer[EDD_MBR_SIG_MAX]`
- `setup_header hdr` (the bzImage header above, copied here)
- 36 bytes padding

### EFI hand-off

`arch/x86/boot/compressed/efi.c` + `arch/x86/boot/startup/efi-mixed.S` define the EFI entry point. The EFI hand-off ABI:

- Entry point at `setup_header.handover_offset` past the kernel base
- Arguments: ImageHandle (RDI), SystemTable (RSI), boot_params (RDX)
- Boot-services-still-active mode: kernel calls `ExitBootServices()` itself
- 32-on-64 mixed mode: 32-bit firmware calling 64-bit kernel via `efi-mixed.S` thunk

### Confidential-compute guest entry

- **AMD SEV-SNP**: `arch/x86/boot/compressed/sev.c` handles VC (#VC) exceptions during early boot; `arch/x86/boot/startup/sev-startup.c` continues post-decompression. The GHCB protocol with the hypervisor is preserved per AMD spec.
- **Intel TDX**: `arch/x86/boot/compressed/tdx.c` + `tdcall.S` issue TDCALL/TDVMCALL hypercalls; `arch/x86/boot/startup/sme.c` handles encryption setup. TDX module ABI per Intel spec.

### kexec hand-off

`arch/x86/kernel/machine_kexec_64.c` orchestrates copying segments to their target addresses (using a "control page" intermediate), disabling interrupts, and jumping to `relocate_kernel_64.S`. The destination kernel sees:

- A `boot_params` constructed from the kexec_load's segment list
- The same architectural state that a fresh boot would see (IDT cleared, paging set to identity-map first 4 MiB, etc.)
- Purgatory has already verified SHA-256 of the new kernel image

The implementation is byte-identical to upstream so an upstream kexec-tools binary can `kexec` into a Rookery kernel and vice versa (REQ-15).

### Purgatory

`arch/x86/purgatory/` is a tiny self-contained binary (no libc, no kernel calls) embedded in the kernel image at build time. Its job: between the source kernel and the destination kernel during kexec, verify SHA-256 of the destination, then jump. Compat: the SHA-256 implementation produces bit-identical hashes; the jump entry-point convention matches upstream.

### AP bringup trampoline

`arch/x86/realmode/init.c` + `arch/x86/realmode/rm/trampoline_64.S`. The BSP issues INIT-SIPI-SIPI to each AP; APs start in real mode at the trampoline; trampoline transitions through protected mode to long mode and joins the running kernel via `secondary_startup_64`. Compat: the trampoline's location, layout, and protocol with the BSP are preserved (matters because the BSP fills in trampoline data structures during boot).

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `boot_params` field reads (assumes valid bootloader contract) | `kani::proofs::arch::x86::boot::params_field_safety` |
| Decompressor output buffer writes | `kani::proofs::arch::x86::boot::decompress_buffer_safety` |
| Identity-map page-table writes | `kani::proofs::arch::x86::boot::ident_map_safety` |
| Trampoline data write (BSP→AP handoff) | `kani::proofs::arch::x86::boot::trampoline_handoff_safety` |
| kexec segment copy | `kani::proofs::arch::x86::kexec::segment_copy_safety` |
| Purgatory SHA-256 buffer access | `kani::proofs::arch::x86::purgatory::sha256_buffer_safety` |

### Layer 2: TLA+ models

Inherited from `arch/x86/00-overview.md`:
- `models/arch/x86/smp_boot.tla` — proves SMP CPU bringup invariants: every AP transitions Real → Protected → Long mode and reaches `cpu_init` exactly once; no two APs collide on the trampoline page. (This Tier-3 doc owns the model.)
- `models/arch/x86/kexec_handoff.tla` — proves the kexec hand-off transitions never leave the destination kernel observing partial state from the source kernel. Liveness: every queued segment is copied before jumping. (This Tier-3 doc owns the model.)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| `setup_header` field offsets | Every named field is at the offset specified in `Documentation/arch/x86/boot.rst` | `kani::proofs::arch::x86::boot::header_layout_invariant` |
| `boot_params` field offsets | Same for `boot_params` | `kani::proofs::arch::x86::boot::params_layout_invariant` |
| Trampoline data | BSP-written fields are read by AP only after IPI delivery | `kani::proofs::arch::x86::boot::trampoline_ordering_invariant` |
| kexec segment list | All segments fit within target address ranges; no overlap | `kani::proofs::arch::x86::kexec::segment_layout_invariant` |

### Layer 4: Functional correctness (opt-in)

- **bzImage header parser** correctness via Creusot — file-format parsing is a well-understood Creusot target.
- **Purgatory SHA-256** correctness via Verus — cross-references `lib/lightweight-crypto.md`'s mandatory Layer-4 proof of libsha256.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | This component's responsibility | Default inherited from |
|---|---|---|
| **KASLR** | `arch/x86/boot/compressed/kaslr.c` analog generates the random kernel base + physical-load address. Default-on per CONFIG_RANDOMIZE_BASE=y. | `00-security-principles.md` § Default-on configurable off (RANDMMAP / ASLR row) |
| **PAGEEXEC** (NX bit) | Early identity-map page tables (`ident_map_64.c` analog) set NX on data + heap regions; only `.text` + early-bootloader-stub get exec | `00-security-principles.md` § Mandatory |
| **KERNEXEC** (kernel W^X) | `vmlinux.lds.S` analog places `.text` + `.rodata` in RX/RO sections; the long-mode page tables enforce | `00-security-principles.md` § Mandatory |
| **CLOSE_KERNEL / CLOSE_USERLAND** (SMEP/SMAP) | CR4 bits set during `head64.c::x86_64_start_kernel` analog, AFTER paging is on | `00-security-principles.md` § Mandatory |
| **DIRECT_CALL** (Spectre-v2 mitigation pre-init) | Early IDT handlers use retpolines from the very first dispatch | `00-security-principles.md` § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT**, **CONSTIFY**: Boot-path data structures are mostly statics; no allocations beyond the early decompression buffer.
- **SIZE_OVERFLOW**: All address arithmetic in the decompressor + identity map uses `checked_add` / `wrapping_add` per the clippy lint.

### Row-2 / GR-RBAC integration

The boot path runs before any LSM is initialized; no LSM hooks fire during boot. GR-RBAC initialization happens later via `security_init` (after `start_kernel`). This component has no LSM-hook surface.

### Userspace-visible behavior changes

- **Exec-gain ELF note recognition**: NOT in this Tier-3. The note is read by `fs/binfmt_elf.c` analog at exec time, not at boot. (See `fs/00-overview.md` § exec-binfmt.md.)
- **MPROTECT default on**: Same — applies post-boot, doesn't affect this component.

### Verification

(See § Verification above.)

