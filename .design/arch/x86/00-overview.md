# Subsystem: arch/x86/ — x86_64 architecture

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - arch/x86/
  - arch/x86/boot/
  - arch/x86/coco/
  - arch/x86/crypto/
  - arch/x86/entry/
  - arch/x86/events/
  - arch/x86/hyperv/
  - arch/x86/ia32/
  - arch/x86/include/
  - arch/x86/kernel/
  - arch/x86/kvm/
  - arch/x86/lib/
  - arch/x86/mm/
  - arch/x86/net/
  - arch/x86/pci/
  - arch/x86/platform/
  - arch/x86/power/
  - arch/x86/purgatory/
  - arch/x86/ras/
  - arch/x86/realmode/
  - arch/x86/virt/
  - arch/x86/xen/
  - arch/x86/Kconfig
  - arch/x86/Makefile
-->

## Summary
Tier-2 overview for the x86_64 architecture port — Rookery's only architecture in v0. Owns every CPU-, MMU-, IRQ-, and platform-specific interaction with x86_64 hardware: boot from the bootloader hand-off through `start_kernel`, syscall and exception entry, page-table management, FPU/SIMD state, kexec hand-off, KVM host support, confidential-compute guest support (SEV-SNP, TDX), virtualized-guest enlightenments (Xen, Hyper-V), CPU-vulnerability mitigations, and the x86 BPF JIT.

This document does NOT design Rust workspace/crate shape (delegated to the implementing instance per `00-overview.md`). It enumerates components, declares the compat-contract slice owned by x86, lists planned Tier-3 component docs, and fixes verification + hardening expectations.

## Upstream references in scope

`arch/x86/` is 25+ subdirectories totaling tens of thousands of lines. The Tier-3 docs spawned from this overview cover them in detail; the table below is a roadmap.

| Upstream subdir | Purpose | Planned Tier-3 doc |
|---|---|---|
| `arch/x86/boot/`         | Real-mode + protected-mode setup before the 64-bit kernel runs; bzImage header (`header.S`), early decompression (`compressed/`) | `boot.md` |
| `arch/x86/realmode/`     | Real-mode trampolines used for AP CPU bringup and S3 wakeup | folded into `boot.md` and `kernel-platform.md` |
| `arch/x86/purgatory/`    | Tiny stub running between kexec'd kernels (SHA-256 image verification) | folded into `boot.md` (kexec section) |
| `arch/x86/entry/`        | Syscall, IRQ, exception entry/exit assembly + C; vDSO; vsyscall (legacy) | `entry.md`, `vdso.md` |
| `arch/x86/kernel/`       | Core x86 platform: head_64.S, setup, traps, IDT, SMP bringup, signals, FPU, time, kexec, CPU init, ftrace, modules | `kernel-platform.md` (umbrella; spawns smaller docs as content grows) |
| `arch/x86/mm/`           | Page tables, page-fault handler, ioremap, PAT, KASLR, KASAN/KMSAN init, AMD SEV memory encryption | `paging.md`, `mem_encrypt.md` |
| `arch/x86/coco/`         | Confidential-compute guest support: SEV-SNP (AMD) + TDX (Intel) infrastructure | `coco.md` |
| `arch/x86/kvm/`          | KVM hypervisor module (host side): VMX (Intel) + SVM (AMD) | `kvm.md` (cross-references `virt/00-overview.md`) |
| `arch/x86/xen/`          | Xen PV/PVH guest enlightenments | `xen-guest.md` |
| `arch/x86/hyperv/`       | Hyper-V guest enlightenments | `hyperv-guest.md` |
| `arch/x86/events/`       | x86 PMU drivers (Intel, AMD, Zhaoxin); the `perf_event` substrate | `pmu.md` (cross-references `kernel/events/`) |
| `arch/x86/crypto/`       | AES-NI, SHA-NI, AVX-assisted hashes/ciphers | `crypto-accel.md` |
| `arch/x86/net/`          | BPF JIT for x86_64 | `bpf-jit.md` |
| `arch/x86/pci/`          | x86 PCI configuration mechanisms (mmconfig, type-1 ports) | `pci.md` |
| `arch/x86/platform/`     | Platform-specific quirks (UV, Intel-MID legacy, OLPC, EFI, IRIS, AMD-CCP) | folded into `kernel-platform.md` |
| `arch/x86/power/`        | Suspend-to-RAM (S3) and hibernate (S4) | `power.md` |
| `arch/x86/ras/`          | Machine-check architecture, CEC (Corrected-Errors Collector) | `ras.md` |
| `arch/x86/virt/`         | x86 virtualization shared helpers (vmx common, svm common) | folded into `kvm.md` |
| `arch/x86/lib/`          | x86-specific libc-equivalents: memcpy, memset, copy_to/from_user, retpoline thunks | folded into `kernel-platform.md` and `entry.md` |
| `arch/x86/ia32/`         | 32-bit-on-64-bit compat layer | folded into `entry.md` |
| `arch/x86/include/asm/`  | x86 architecture headers (339 files) | (covered indirectly by all the above; major headers cited in their respective Tier-3 docs) |

The 22 architectures other than `x86` are out of scope for v0 (per `arch/00-overview.md`).

## Compatibility contract

x86_64 owns the following slice of the project's full-ABI drop-in compat surface (`00-overview.md` REQ-1 through REQ-5):

### Boot interface

The x86 boot protocol (`Documentation/arch/x86/boot.rst`, `arch/x86/boot/header.S`, `arch/x86/include/uapi/asm/bootparam.h`) defines how the bootloader hands control to the kernel.

- **bzImage header**: byte-identical layout. Magic `0x53726448` ("HdrS"), `setup_sects`, `version`, `realmode_swtch`, `start_sys_seg`, `kernel_version`, `type_of_loader`, `loadflags`, `setup_move_size`, `code32_start`, `ramdisk_image`, `ramdisk_size`, `heap_end_ptr`, `cmd_line_ptr`, `init_size`, `handover_offset`, `kernel_info_offset`. Owned by `boot.md`.
- **boot_params struct**: byte-identical, including `screen_info`, `apm_bios_info`, `tboot_addr`, `ist_info`, `acpi_rsdp_addr`, `e820_table`, `eddbuf`, `efi_info`, command-line tail. Owned by `boot.md`.
- **EFI hand-off**: identical EFI hand-off entry point and parameter layout (`handover_offset`, `efi_info` fields, EFI memory map relayed via `boot_params`). Owned by `boot.md`.
- **kexec hand-off**: the kernel produces a kexec hand-off byte-identical to upstream's (REQ-15 / D5). Source: `arch/x86/kernel/machine_kexec_64.c`, `arch/x86/kernel/relocate_kernel_64.S`, `arch/x86/purgatory/`. Owned by `boot.md`.
- **xen PVH boot entry**: identical PVH note + entry contract for Xen PVH boot. Owned by `xen-guest.md`.

### Syscall interface

x86_64 syscall ABI per the System V x86-64 calling convention (subset for syscalls):

- **Instructions**: `syscall` (preferred), `int 0x80` (legacy 32-bit-on-64-bit), `sysenter` (legacy compat). Identical instruction recognition.
- **Registers in (x86_64)**: `rax`=syscall number, `rdi`=arg0, `rsi`=arg1, `rdx`=arg2, `r10`=arg3, `r8`=arg4, `r9`=arg5. (Note `r10` not `rcx`; `rcx` is clobbered by the `syscall` insn for `rip`.)
- **Registers out**: `rax`=result. Negative values in `[-4095, -1]` indicate `-errno` per upstream convention.
- **Clobbered**: `rcx` (saved `rip`), `r11` (saved `rflags`).
- **Syscall numbers**: identical to `arch/x86/entry/syscalls/syscall_64.tbl` (421 entries at baseline including `x32` overlapping range).
- **vDSO fast paths**: `__vdso_clock_gettime`, `__vdso_clock_getres`, `__vdso_gettimeofday`, `__vdso_time`, `__vdso_getcpu` exposed via `linux-vdso.so.1` mapped into every userspace process. Owned by `vdso.md`.
- **vsyscall page** (legacy emulation): `0xffffffffff600000` virtual address with `gettimeofday`, `time`, `getcpu` stubs. CONFIG_LEGACY_VSYSCALL_* gated; `EMULATE` mode is the upstream default. Owned by `vdso.md`.
- **Compat layer**: 32-bit-on-64-bit syscalls (`int 0x80`, `syscall32` via `entry_64_compat.S`) consume the 32-bit `syscall_32.tbl`. Owned by `entry.md`.
- **FRED entry**: optional path via Flexible Return and Event Delivery (`entry_64_fred.S`, `entry_fred.c`) on FRED-capable CPUs; for Rookery, FRED support tracks upstream's CONFIG_X86_FRED. Owned by `entry.md`.

### CPU state visible to userspace

- **General-purpose registers**: 16 GPRs (`rax`-`r15`) saved in `pt_regs` (`arch/x86/include/asm/ptrace.h`); ptrace `NT_PRSTATUS` register set is byte-identical.
- **Segment registers**: `cs`, `ss`, `ds`, `es`, `fs`, `gs` saved in `pt_regs`. `fs.base`, `gs.base` saved per-task (used by TLS).
- **FPU/SIMD state**: stored in `struct fpu` (variable-size, per `XSAVE`/`XSAVES` features advertised by CPUID). Layout matches upstream's `fpu_state_perm` per-task tracking. Owned by `kernel-platform.md` (FPU section).
- **Signal frame**: `struct ucontext_t` and `struct sigcontext` from `include/uapi/asm/sigcontext.h` are byte-identical. The signal-handler entry restores GPRs, segment registers, FPU state from the frame on `sigreturn(2)`. Owned by `signal.md`.
- **TLS**: `arch_prctl(ARCH_SET_FS, ...)`/`ARCH_GET_FS` semantics + the four-slot user-segment TLS table (legacy 32-bit). Owned by `entry.md` plus `kernel-platform.md`.

### Memory layout visible to userspace

- **Userspace virtual address range** (`TASK_SIZE`): on 4-level paging, `0x00007fff_ffffffff` (47-bit). On 5-level paging (CONFIG_X86_5LEVEL + CR4.LA57), `0x00ffffff_ffffffff` (56-bit). Identical to upstream.
- **mmap base randomization**: `arch_mmap_rnd_bits` range `[28, 32]`; identical default behavior.
- **Page sizes**: 4 KiB (4 K), 2 MiB (PMD-large), 1 GiB (PUD-huge). All advertised to userspace via `/proc/<pid>/smaps` `KernelPageSize`/`MMUPageSize`.
- **`/proc/<pid>/maps` and `/proc/<pid>/smaps`**: format byte-identical (compat-critical for tooling: gdb, valgrind, strace, perf, libunwind).
- **`/proc/cpuinfo`**: identical fields and ordering (`vendor_id`, `cpu family`, `model`, `model name`, `stepping`, `microcode`, `cpu MHz`, `cache size`, `physical id`, `siblings`, `core id`, `cpu cores`, `apicid`, `initial apicid`, `fpu`, `fpu_exception`, `cpuid level`, `wp`, `flags`, `bugs`, `bogomips`, `clflush size`, `cache_alignment`, `address sizes`, `power management`). Owned by `cpuinfo.md`.

### ELF and binfmt

- **e_machine = `EM_X86_64` (62)** for 64-bit; `EM_386` (3) for 32-bit compat.
- **ELF auxiliary vector entries**: identical (`AT_PLATFORM`, `AT_BASE_PLATFORM`, `AT_HWCAP`, `AT_HWCAP2`, `AT_RANDOM`, `AT_SYSINFO_EHDR` pointing at vDSO).
- **HWCAP / HWCAP2** bit definitions: identical per `arch/x86/include/uapi/asm/hwcap.h` (where present) and `arch/x86/include/asm/cpufeatures.h`. Owned by `cpuinfo.md`.

### Hypervisor / virtualization interfaces

- **KVM userspace ABI** (`/dev/kvm` ioctls per `include/uapi/linux/kvm.h`) — owned at the cross-arch level by `virt/00-overview.md`; x86-specific KVM_GET_SUPPORTED_CPUID, KVM_SET_TSC_KHZ, MSR-based ops are owned here. Owned by `kvm.md`.
- **Xen PV/PVH guest interface** — hypercall ABI, event channels, grant tables (Xen-defined; we implement the guest side). Owned by `xen-guest.md`.
- **Hyper-V TLFS-defined synthetic MSRs and hypercalls** — guest-side. Owned by `hyperv-guest.md`.
- **Confidential-guest interfaces**: AMD SEV-SNP `GHCB` protocol, Intel TDX `TDCALL`/`TDVMCALL`. Owned by `coco.md`.

### IDT / IRQ / exception interface (mostly internal but ptrace-visible)

- **Hardware exception vectors 0-31** are fixed by the CPU; their handlers' visible behavior (signal delivery, ptrace stops, core-dump generation, `/proc/<pid>/syscall` semantics) is byte-identical.
- **Exception → signal mapping**: `#PF` → `SIGSEGV` or `SIGBUS`, `#GP` → `SIGSEGV`, `#UD` → `SIGILL`, `#DE` → `SIGFPE`, `#OF` → `SIGSEGV`, etc. Owned by `entry.md`.

### MSRs and CPUID

- **Userspace-visible MSRs via `/dev/cpu/<n>/msr`**: identical per upstream `arch/x86/kernel/msr.c`.
- **CPUID exposure to userspace via `/dev/cpu/<n>/cpuid`**: identical per upstream `arch/x86/kernel/cpuid.c`.
- **rdtsc / rdtscp / rdpmc** behavior: identical, including conditional disables via `prctl(PR_SET_TSC, ...)` / `cr4.TSD`.

## Requirements

- REQ-1: A Rookery-built `bzImage` consumes the upstream-defined boot protocol (`Documentation/arch/x86/boot.rst`) byte-for-byte: the same `boot_params` fields, the same `setup_header` layout, the same EFI hand-off conventions. (Drop-in compat source.)
- REQ-2: A Rookery kernel produces a kexec hand-off (`segment[]` + `boot_params`) byte-identical to upstream's such that an existing `kexec`-tools userspace (or a Rookery-built one) can `kexec` to and from a Rookery kernel.
- REQ-3: The x86_64 syscall ABI (registers, `syscall`/`int 0x80`/`sysenter` instruction handling, syscall numbers per `syscall_64.tbl`) is byte-identical to upstream. Per-syscall semantics live in Tier-5 `uapi/syscalls/*.md`.
- REQ-4: Userspace-visible CPU state via `pt_regs`, `struct sigcontext`, `struct ucontext_t`, ptrace register sets (`NT_PRSTATUS`, `NT_PRFPREG`, `NT_X86_XSTATE`, `NT_X86_SHSTK`, etc.) is byte-identical.
- REQ-5: Page-table layout is identical: 4-level (CR4.LA57=0) or 5-level (CR4.LA57=1) paging with the same page sizes (4 KiB / 2 MiB / 1 GiB) and the same userspace virtual-address range bounds.
- REQ-6: vDSO `linux-vdso.so.1` exports byte-identical symbols (`__vdso_clock_gettime`, `__vdso_clock_getres`, `__vdso_gettimeofday`, `__vdso_time`, `__vdso_getcpu`) and the legacy `vsyscall` page (CONFIG_LEGACY_VSYSCALL_EMULATE) when configured.
- REQ-7: `/proc/cpuinfo` content format is byte-identical to upstream — fields, ordering, units.
- REQ-8: KVM, Xen-guest, Hyper-V-guest, SEV-SNP-guest, TDX-guest interfaces preserve the user-visible ABI (`/dev/kvm` ioctls; hypercall ABIs are dictated by external specs).
- REQ-9: x86-specific compat layer (`arch/x86/ia32/`) preserves `int 0x80` and 32-bit `syscall_32.tbl` numbers so existing 32-bit userspace continues to work.
- REQ-10: x86 BPF JIT (in `arch/x86/net/`) produces verified-equivalent code for any BPF program upstream's JIT accepts; behavior on rejection cases matches upstream.
- REQ-11: x86 cryptographic accelerators (AES-NI, SHA-NI, AVX-assisted) integrate via the kernel crypto API (see `crypto/00-overview.md`); the user-visible algorithms registered are identical (priorities and names).
- REQ-12: All CPU-vulnerability mitigations upstream supports at the baseline commit (Spectre-v1/v2, Meltdown, MDS, TAA, SRBDS, MMIO Stale Data, retbleed, GDS, BHI, SLS, EntryBleed, RFDS, Reg-File Data Sampling, …) are implemented, with identical `cpu_show_*()` reporting under `/sys/devices/system/cpu/vulnerabilities/`.
- REQ-13: All Tier-3 component docs spawned from this overview are listed in the Architecture section's Layout map.

## Acceptance Criteria

- [ ] AC-1: A Rookery-built `bzImage` boots under QEMU using a stock GRUB or systemd-boot configuration without modification. Verified by booting through `start_kernel` to userspace `init`. (covers REQ-1)
- [ ] AC-2: A Rookery kernel can `kexec` into an upstream Linux kernel and vice versa, verified by an integration-test harness. (covers REQ-2)
- [ ] AC-3: Every entry in `arch/x86/entry/syscalls/syscall_64.tbl` resolves to a Rookery handler that is byte-equivalent in entry/exit register conventions to upstream's, verified by a `strace`-based golden-trace comparison. (covers REQ-3)
- [ ] AC-4: A `ptrace`-based test harness reads `NT_PRSTATUS`, `NT_PRFPREG`, `NT_X86_XSTATE` from a tracee under both Rookery and upstream and reports byte-identical buffers. (covers REQ-4)
- [ ] AC-5: A test booting with both `nopti nokaslr` and the full default mitigations produces matching `/proc/<pid>/maps` ranges between Rookery and upstream for an unprivileged ELF. (covers REQ-5)
- [ ] AC-6: `nm` over Rookery's vDSO `.so` and upstream's vDSO `.so` reports the same symbol set; an `LD_PRELOAD` test invoking each vDSO symbol returns identical results to a baseline. (covers REQ-6)
- [ ] AC-7: A diff between Rookery's `/proc/cpuinfo` and upstream's on the same hardware shows zero non-ignorable differences (microcode revision and bogomips noise excepted). (covers REQ-7)
- [ ] AC-8: KVM `kvm-unit-tests` passes against Rookery `KVM=y` with the same pass/fail set as upstream. (covers REQ-8)
- [ ] AC-9: A 32-bit `i386` userspace binary (`/usr/bin/file` reports `ELF 32-bit LSB executable, Intel 80386`) executes correctly under Rookery's 64-bit kernel. (covers REQ-9)
- [ ] AC-10: BPF selftest suite passes under Rookery x86_64 with the same pass/fail set as upstream. (covers REQ-10)
- [ ] AC-11: `cryptsetup`, `wireguard`, `dm-integrity`, and any other AES-NI / SHA-NI consumer in userspace runs successfully against Rookery. (covers REQ-11)
- [ ] AC-12: `/sys/devices/system/cpu/vulnerabilities/*` files exist and report the same `Mitigation: ...` strings as upstream on identical hardware. (covers REQ-12)
- [ ] AC-13: This document's Architecture section's Layout map enumerates Tier-3 docs covering boot, entry, paging, kernel-platform (CPU bringup / FPU / signal / time / IDT / TSS / kexec), vDSO, cpuinfo, ptrace ABI, ELF, CPU mitigations, COCO, hyperv-guest, xen-guest, KVM, PCI, RAS, power, crypto-accel, BPF JIT. (covers REQ-13)

## Architecture

### Layout map (Tier-3 docs spawned from this overview)

```
.design/arch/x86/
  00-overview.md          ← this document
  boot.md                 ← bzImage header, decompression, EFI hand-off, kexec hand-off, real-mode trampolines, purgatory
  entry.md                ← syscall entry/exit (entry_64.S, FRED), 32-bit compat (entry_64_compat.S), IRQ entry, exception delivery, ia32
  vdso.md                 ← vDSO contents, vvar/vclock pages, vsyscall legacy emulation
  paging.md               ← 4-level/5-level page tables, page-fault handler, ioremap, PAT, KASLR, KASAN/KMSAN init
  mem_encrypt.md          ← AMD SEV/SME memory encryption (host side, complementary to coco.md guest side)
  kernel-platform.md      ← head_64.S, head64.c, setup, traps, IDT, TSS, GDT, smpboot, FPU, time/tsc/hpet/rtc, ftrace, modules, paravirt
  signal.md               ← signal frame layout, sigreturn, restorer, signal delivery on x86_64 + ia32
  cpuinfo.md              ← /proc/cpuinfo format, CPUID exposure (HWCAP, AT_HWCAP, AT_HWCAP2)
  ptrace-abi.md           ← ptrace request set on x86_64, NT_PRSTATUS/NT_PRFPREG/NT_X86_XSTATE/NT_X86_SHSTK formats
  elf.md                  ← x86_64 ELF binfmt specifics, e_machine, vDSO mapping, AT_SYSINFO_EHDR
  cpu-mitigations.md      ← Spectre/Meltdown/MDS/retbleed/GDS/BHI/.../EntryBleed/RFDS  retpolines, IBPB, IBRS, RSB, IBT, CET, LASS
  coco.md                 ← SEV-SNP guest, TDX guest, GHCB protocol, TDCALL/TDVMCALL
  hyperv-guest.md         ← Hyper-V synthetic MSRs, hypercalls, VMBus shim
  xen-guest.md            ← Xen PV/PVH/HVM guest interface
  kvm.md                  ← KVM host: VMX (Intel) + SVM (AMD), MMU, lapic emulation, nested virt
  pci.md                  ← x86 PCI cfg-space mechanisms (mmconfig, type-1)
  pmu.md                  ← x86 PMU drivers (Intel core PMU, AMD core PMU, uncore PMUs)
  crypto-accel.md         ← AES-NI, SHA-NI, AVX-assisted ciphers/hashes; cross-references crypto/00-overview.md
  bpf-jit.md              ← x86_64 BPF JIT
  power.md                ← suspend-to-RAM (S3), hibernate (S4), CPU-idle drivers
  ras.md                  ← machine-check architecture, mce_amd, mce_intel, CEC
```

(Tier-3 docs not yet authored; this overview commits to their existence.)

### Cross-references

- Verification stack: `00-rust-conventions.md` § Verification stack (Layers 1-4)
- Compat surface: `00-overview.md` § Compatibility contract enumeration
- Glossary: `00-glossary.md` x86_64-specific section (MSR, CR0/CR2/CR3/CR4, IDT, TSS, KASLR)
- Hardening: `references/grsec-pax-notes.md` for the catalog; binding policy in deferred `00-security-principles.md`

### Rust module organization (informative)

Tier-3 docs name specific Rust module paths in `kernel::arch::x86::*`. The arch-x86 crate(s) live at the kernel-source-tree's `arch/x86/rust/` (analog to the existing top-level `rust/` for the cross-arch kernel crate). Naming conventions per `00-rust-conventions.md`:

- `kernel::arch::x86::boot` ↔ `arch/x86/boot/`
- `kernel::arch::x86::entry` ↔ `arch/x86/entry/`
- `kernel::arch::x86::paging` ↔ `arch/x86/mm/` (page-table portions)
- `kernel::arch::x86::idt` ↔ `arch/x86/kernel/idt.c`, `arch/x86/include/asm/desc.h`
- `kernel::arch::x86::cpu` ↔ `arch/x86/kernel/cpu/`
- `kernel::arch::x86::fpu` ↔ `arch/x86/kernel/fpu/`
- `kernel::arch::x86::signal` ↔ `arch/x86/kernel/signal_64.c`, `signal.c`
- `kernel::arch::x86::msr` ↔ `arch/x86/include/asm/msr.h`
- `kernel::arch::x86::kvm` ↔ `arch/x86/kvm/`
- `kernel::arch::x86::xen` ↔ `arch/x86/xen/`
- `kernel::arch::x86::hyperv` ↔ `arch/x86/hyperv/`
- `kernel::arch::x86::coco` ↔ `arch/x86/coco/`
- `kernel::arch::x86::bpf_jit` ↔ `arch/x86/net/`

(The implementing instance may further decompose; these names are reference, not binding.)

### Locking and concurrency

x86-specific code holds locks in two predominant places:
1. **IDT installation** during boot is done with all CPUs at the BSP barrier; no runtime mutex is needed.
2. **Per-CPU APIC manipulation** uses `local_irq_save` / `local_irq_restore` (i.e., disables IRQs) to serialize against IPI delivery. Encoded as a `kernel::arch::x86::IrqGuard` RAII type.

The vDSO update path (clock-source updates) uses a seqlock between writers (timekeeping interrupt) and readers (userspace vDSO callers). TLA+ model required (Layer 2): proves that vDSO readers never observe torn writes. This is a cross-arch concern but x86 has a concrete instance.

### Error handling

x86 entry paths must NOT return `Result` (exceptions and IRQs do not have a fallible-return contract). Errors detected during entry (bad IRET frame, invalid syscall number) translate to a signal delivery to the offending task, not a return value. The Rust-side type is `kernel::arch::x86::EntryOutcome` with variants `Continue`, `DeliverSignal { sig, info }`, `BugOn`. The `BugOn` arm panics via `kernel::panic!` (not bare `panic!`).

## Verification

Per `00-overview.md` D4, this Tier-2 document declares the verification expectations for the x86 subsystem. Each Tier-3 doc enumerates its concrete artifacts; this section sets the floor.

### Layer 1: Kani SAFETY proofs

Anticipated `unsafe` blocks across arch/x86 (counting only logical clusters, not literal block count):

- IDT manipulation, GDT manipulation, TSS write — `kernel::arch::x86::idt::*`, `gdt::*`, `tss::*`. Harness names: `kani::proofs::arch::x86::idt::*_safety`.
- MSR read/write — `kernel::arch::x86::msr::{rdmsr,wrmsr}`. Harness: `kani::proofs::arch::x86::msr::access_safety`.
- Page-table walking and modification — `kernel::arch::x86::paging::*`. Harnesses: per-level walk safety (4 each for 4-level paging, 5 each for 5-level).
- copy_to/from_user assembly intrinsics — `kernel::arch::x86::uaccess::*`. Harness: `kani::proofs::arch::x86::uaccess::copy_safety`.
- `iretq` / SYSRET return assembly — `kernel::arch::x86::entry::return_to_user`. Harness asserts no kernel-state leak in any branch.
- FPU XSAVE/XRSTOR — `kernel::arch::x86::fpu::*`. Harness: state-buffer-bounds safety.
- vDSO data-page mapping — `kernel::arch::x86::vdso::*`. Harness: shared-mapping safety.
- IRQ entry assembly — `kernel::arch::x86::entry::irq_entry`. Harness: stack-frame integrity invariant.

### Layer 2: TLA+ models

Models required at this tier:

- `models/arch/x86/vdso_seqlock.tla` — proves vDSO clock-source updates never expose torn-write states to readers. Safety + liveness.
- `models/arch/x86/smp_boot.tla` — proves SMP CPU bringup invariants: every AP transitions Real → Protected → Long mode and reaches `cpu_init` exactly once; no two APs collide on the trampoline page. Safety only (liveness conditional on each AP receiving INIT/SIPI).
- `models/arch/x86/idt_install.tla` — proves IDT entry installation is atomic w.r.t. concurrent interrupt delivery (BSP-only constraint at boot; runtime updates restricted to one writer with IRQs off).
- `models/arch/x86/kexec_handoff.tla` — proves the kexec hand-off transitions never leave the destination kernel observing partial state from the source kernel. Liveness: every queued segment is copied before jumping.

Tier-3 docs add models for their own primitives (e.g., `arch/x86/cpu-mitigations.md` may add a model for the IBPB-on-context-switch invariant).

### Layer 3: Kani harnesses for data-structure invariants

x86-tier data structures with mandatory invariant harnesses:

- **Page-table walker** — every level's `walk_*_entry()` function carries an invariant harness asserting (a) the entry's physical address resolves to a valid page-table page, (b) the present bit is consistent with PFN reachability, (c) the access/dirty bits never decrease except through explicit clear paths.
- **IDT entry table** — invariant harness asserting every vector 0–31 maps to its CPU-architecture-fixed handler kind (trap/interrupt/abort), every vector 32–255 is either unset or maps to a registered IRQ handler.
- **TSS layout** — invariant harness asserting each per-CPU TSS has its IST stacks within the per-CPU stack area and never overlaps the kernel direct-map.
- **per-CPU offset table** — once `kernel::arch::x86::cpu::PerCpu<T>` is authored (issue #4), invariant harness asserting per-CPU access never crosses the per-CPU area boundary.

### Layer 4: Functional correctness via Creusot / Verus / Prusti (opt-in)

Subsystem opt-ins anticipated:

- `arch/x86/crypto-accel.md` — AES-NI / SHA-NI primitives are candidates for **Verus** functional-correctness proofs (the underlying instructions are deterministic; "this Rust function computes AES-128 of the input" is a tractable Verus theorem).
- `arch/x86/bpf-jit.md` — JIT correctness (the emitted x86 bytes execute the BPF program's semantics) is a candidate for **Creusot**, but is large enough that it may slip past v0; track but do not require.
- `arch/x86/cpu-mitigations.md` — retpoline-thunk equivalence (the thunk is observably indistinguishable from a direct indirect call in the absence of speculation) is a candidate for hand-written assembly proof rather than a Rust verifier.

## Hardening

Placeholder per `00-overview.md` D6 (binding policy in deferred `00-security-principles.md`). The arch/x86 tier owns the *implementation* of the following hardening features; their policy gating lands in the principles doc once authored.

- KASLR (`arch/x86/boot/compressed/kaslr.c`, `arch/x86/mm/kaslr.c`)
- KPTI (Page-Table Isolation, Meltdown mitigation; `arch/x86/mm/pti.c`)
- SMEP, SMAP (CR4 bits enabled at boot per CPU capability)
- Intel CET (Shadow Stack via `IA32_S_CET` MSR, IBT via `IA32_U_CET`)
- IBT (Indirect Branch Tracking) for kernel via CFI
- Retpolines, IBPB, IBRS, eIBRS (per `arch/x86/include/asm/nospec-branch.h`)
- LASS (Linear Address Space Separation) where supported
- PRIVATE_KSTACKS-equivalent: per-CPU IST stacks already isolated; per-task kernel stacks independent
- RANDKSTACK-equivalent: optional kernel-stack offset randomization on syscall entry
- Memory encryption: SME, SEV (host); SEV-SNP, TDX (guest in `coco.md`)

Each Tier-3 doc whose content touches a hardening feature gains a "Hardening" section.

## Resolved Decisions

### D1: 32-bit-only kernel (`X86_32=y`) is out of scope
Drop-in compat targets prevalent x86_64 distros. The 32-bit *userspace* compat layer (`arch/x86/ia32/`) IS in v0 scope; a pure 32-bit *kernel* is not. No `i386_kernel.md` design doc is authored in v0; one may be added as a deferred placeholder if future demand arises.

### D2: FRED implemented alongside classic IDT path in v0
Both code paths are required because the bootloader hands off in IDT mode and FRED must be enabled mid-boot if available. An FRED-only kernel cannot boot on non-FRED hardware, so the dual-path implementation is mandatory. `entry.md` (Tier 3) covers both.

### D3: Full PV-OPS paravirt layer preserved in v0
Hypervisor-guest performance is part of the drop-in promise — if Xen / Hyper-V / KVM guests run noticeably slower under Rookery, that breaks user expectations even if functionally correct. `kernel-platform.md` (Tier 3) covers paravirt dispatch. Tier-3 work for `xen-guest.md`, `hyperv-guest.md`, and `kvm.md` (host) all consume the paravirt dispatch layer.

### D4: math-emu out of scope for v0
Every x86_64 CPU has hardware FPU mandated by the architecture. `arch/x86/math-emu/` is not reverse-engineered.

### D5: vsyscall page in EMULATE mode by default
Matches upstream default (`CONFIG_LEGACY_VSYSCALL_EMULATE=y`). Preserves compat with very old binaries that still call into the fixed `0xffffffffff600000` mapping. `vdso.md` (Tier 3) owns the full spec.

### D6: KVM is in v0 scope with staged delivery
KVM (`arch/x86/kvm/`) IS in v0. The implementing instance may stage delivery: (a) basic VMX guest support → (b) MSR/CPUID emulation → (c) MMU virtualization → (d) lapic/IOAPIC emulation → (e) nested virt. **v0 graduation bar: `kvm-unit-tests` passes against Rookery KVM with the same pass/fail set as upstream.** The `kvm.md` (Tier 3) doc owns the design and names the staging gates.

## Open Questions

(none — all open questions for this subsystem document are resolved above)

## Out of Scope

- 32-bit-only kernel (`X86_32=y`) — per Q1 recommendation.
- math-emu (`arch/x86/math-emu/`) — per Q4 recommendation.
- Architectures other than x86_64 (every `arch/<name>/` for `<name> != x86`) — per `00-overview.md` D5.
- User Mode Linux (`arch/x86/um/`, `arch/um/`) — per `arch/00-overview.md` Q2.
- `arch/x86/video/` BIOS-VGA legacy — modern systems boot via EFI framebuffer; legacy VGA is deferred to v1+ if demand arises.
- 32-bit `arch/x86/ia32/` syscalls 0–280 (the gap range that no current 32-bit userspace uses) — covered by inclusion of full `syscall_32.tbl` but no per-syscall designs in Phase D.
- Implementation code — `.design/` contains specs only.
