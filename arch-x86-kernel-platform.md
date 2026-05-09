---
title: "Tier-3: arch/x86/kernel-platform — CPU bringup, IDT/TSS, FPU, time"
tags: ["design-doc", "tier-3", "arch-x86"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the x86_64 platform substrate that runs *between* boot (`arch/x86/boot.md`) and the cross-arch core. Owns CPU vendor identification + bringup, GDT/TSS/IDT installation per CPU, FPU/SIMD save-state machinery, MSR access, time-keeping (TSC, HPET, RTC, PIT), local + I/O APIC, runtime code patching (jump_label + static_call + alternative), kernel-module relocation, paravirt_ops dispatch, Intel CET shadow stack, kCFI implementation, dumpstack helpers, and early printk.

This is the largest single x86 component (135+ files in `arch/x86/kernel/` excluding entry/). Its responsibilities are heterogeneous; the doc enumerates each cleanly and routes detail to per-component sub-areas.

### Requirements

- REQ-1: Per-CPU vendor identification works for all upstream-supported vendors (Intel, AMD, Hygon, Zhaoxin, Centaur, Cyrix, Transmeta).
- REQ-2: SMP bringup brings every AP from real-mode trampoline to long mode and registers each in the running kernel. Topology detection (cores/threads/packages) is correct on every supported topology.
- REQ-3: GDT, IDT, TSS layouts are byte-identical to upstream; per-CPU instances at correct addresses in cpu_entry_area.
- REQ-4: FPU/SIMD state save/restore via XSAVE/XRSTOR uses the upstream-defined xstate_feature_size + xstate_offsets; per-task `fpu` struct layout matches upstream.
- REQ-5: MSR access via `wrmsrl_safe` / `rdmsrl_safe` exposes byte-identical semantics; `/dev/cpu/<n>/msr` ABI preserved.
- REQ-6: CPUID access via `__cpuid_count` is byte-identical; `/dev/cpu/<n>/cpuid` ABI preserved.
- REQ-7: HWCAP/HWCAP2 bit definitions match upstream so `getauxval(AT_HWCAP)` returns the same bits.
- REQ-8: Time-keeping matches upstream: TSC calibration accuracy, HPET fallback, RTC boot read; `/proc/timer_list` content format-identical.
- REQ-9: APIC (LAPIC + I/O APIC + x2APIC + UV) drivers each match upstream; vector allocation + IPI delivery byte-identical.
- REQ-10: Runtime code patching (jump_label + static_call + alternative) match upstream's update-protocol; `text_poke_*` API preserved.
- REQ-11: Microcode loading via `/dev/cpu/<n>/microcode` and `/lib/firmware/intel-ucode/` + `/lib/firmware/amd-ucode/` paths preserved.
- REQ-12: MTRR userspace ABI (`/proc/mtrr` content format) preserved.
- REQ-13: kCFI (forward-edge CFI for indirect calls) default-on per `00-security-principles.md`.
- REQ-14: Intel CET shadow stack + IBT default-on when CPU supports per `00-security-principles.md`.
- REQ-15: Stack unwinder (ORC) produces identical traces; `dump_stack` output format identical (matters for crash-decoding tools).
- REQ-16: Paravirt PV-OPS layer preserved per `arch/x86/00-overview.md` D3; Xen / Hyper-V / KVM-guest enlightenment paths function identically.
- REQ-17: Kernel-module relocation (`arch/x86/kernel/module.c`) supports the same relocation types upstream supports, in the same order; existing .ko files load unchanged.
- REQ-18: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `cat /proc/cpuinfo` byte-identical (modulo bogomips noise + microcode revision) on identical hardware. (covers REQ-1, REQ-7)
- [ ] AC-2: `nproc` reports identical CPU count after SMP bringup; `lscpu` topology-tree matches upstream. (covers REQ-2)
- [ ] AC-3: `pahole` of `struct gdt_page`, `struct tss_struct`, `idt_table` produces byte-identical layout. (covers REQ-3)
- [ ] AC-4: AVX/AVX-512/AMX-state preservation across signal handlers test passes; `pahole struct fpu` matches upstream. (covers REQ-4)
- [ ] AC-5: `rdmsr 0x1B` (IA32_APIC_BASE) returns the same value via /dev/cpu/N/msr on Rookery and upstream. (covers REQ-5)
- [ ] AC-6: A test reading every CPUID leaf produces identical results vs. upstream on the same hardware. (covers REQ-6)
- [ ] AC-7: `getauxval(AT_HWCAP)` test program returns identical values. (covers REQ-7)
- [ ] AC-8: `clock_gettime` accuracy test (1M samples) shows ≤ 1µs drift between Rookery and upstream over 60 seconds. (covers REQ-8)
- [ ] AC-9: APIC vector allocation test exercises 100 device IRQ registrations and produces identical vector layout. (covers REQ-9)
- [ ] AC-10: A kernel module using `static_branch_*` toggles correctly; alternative-patched code site executes the patched form when the feature bit is set. (covers REQ-10)
- [ ] AC-11: `iucode_tool -K /lib/firmware/intel-ucode/` updates microcode on a Rookery boot identically. (covers REQ-11)
- [ ] AC-12: `cat /proc/mtrr` produces format-identical output. (covers REQ-12)
- [ ] AC-13: A binary built with `-fcf-protection=full` runs successfully on Rookery; CET selftests under `tools/testing/selftests/x86/` pass. (covers REQ-13, REQ-14)
- [ ] AC-14: `bpftrace -e 'kprobe:__do_*'` traces match expected stack frames; `crash` against a Rookery kernel decodes the call stack correctly. (covers REQ-15)
- [ ] AC-15: A Xen PV guest boots successfully; a Hyper-V guest boots successfully (cross-ref `arch/x86/xen-guest.md` and `arch/x86/hyperv-guest.md`). (covers REQ-16)
- [ ] AC-16: A canonical kernel module (`drivers/misc/lkdtm`) builds + loads on Rookery's vmlinux. (covers REQ-17)
- [ ] AC-17: Hardening section present and follows template. (covers REQ-18)

### Architecture

### Sub-area decomposition

This Tier-3 spawns sub-areas via internal modules; each could become its own Tier-3 doc if/when scope demands. Current scope keeps them in this single doc:

- **CPU init** (`kernel::arch::x86::cpu`): vendor dispatch, per-vendor init, microcode, MTRR, cache info, topology
- **GDT/IDT/TSS** (`kernel::arch::x86::desc`): descriptor tables setup
- **SMP** (`kernel::arch::x86::smp`): CPU bringup, IPI, MSR-based topology
- **APIC** (`kernel::arch::x86::apic`): LAPIC + IO-APIC + x2APIC + IPI
- **FPU** (`kernel::arch::x86::fpu`): XSAVE/XRSTOR machinery
- **Time** (`kernel::arch::x86::time`): TSC + HPET + RTC + sched_clock backend
- **MSR + CPUID** (`kernel::arch::x86::msr`, `cpuid`): low-level instruction wrappers
- **Patching** (`kernel::arch::x86::patching`): jump_label + static_call + alternative
- **Module loader** (`kernel::arch::x86::module`): x86 relocation
- **Paravirt** (`kernel::arch::x86::paravirt`): PV-OPS dispatch
- **CET / kCFI** (`kernel::arch::x86::cet`, `cfi`): control-flow integrity
- **Unwinder** (`kernel::arch::x86::unwind`): ORC + frame + guess
- **Dumpstack** (`kernel::arch::x86::dumpstack`): stack-trace formatting

### Key data structures (Rust mirrors)

- `gdt_page` — `#[repr(C)]` 4-KiB page holding the GDT
- `tss_struct` — `#[repr(C)]` per-CPU TSS with IST stacks
- `idt_table` — array of 256 IDT descriptors per CPU
- `cpuinfo_x86` — per-CPU info struct (`/proc/cpuinfo` data source)
- `fpu` — per-task FPU save area (variable-size based on XSAVE feature mask)
- `clocksource_x86` — TSC / HPET / PIT abstraction
- `apic_driver` — vtable for LAPIC mode (flat / phys / x2apic / UV)

### Locking and concurrency

- BSP runs single-CPU until `smp_init`; AP bringup serialized via `smp_ops.smp_prepare_cpus` and the CPU hotplug state machine
- Per-CPU data accessed via `__this_cpu_read` / `__this_cpu_write` (preempt-safe wrappers)
- Microcode update: stop_machine to halt all CPUs during the update
- Cross-CPU function calls: `smp_call_function_*` IPIs
- Time-keeping: seqlock-protected reader-writer (cross-ref `lib/00-overview.md` § vdso-core.md)

### Error handling

`Result<T, kernel::error::Error>` for fallible ops:
- `Err(ENODEV)` — APIC not detected; HPET unavailable
- `Err(EINVAL)` — bad MSR / CPUID args
- `Err(EBUSY)` — concurrent microcode update; AP already up
- `Err(ENOMEM)` — per-CPU allocation failure

### Out of Scope

- 32-bit `process.c` (not `process_64.c`) — out per `arch/x86/00-overview.md` D1
- Per-CPU mitigation policy (Spectre-v2 selection algorithm) — owned by `arch/x86/cpu-mitigations.md` Tier 3
- Per-platform quirks for legacy hardware (Intel-MID, OLPC, etc.) — folded into platform/* under `arch/x86/00-overview.md`
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| CPU vendor identification + per-vendor init | `arch/x86/kernel/cpu/common.c`, `cpu/intel.c`, `cpu/amd.c`, `cpu/centaur.c`, `cpu/cyrix.c`, `cpu/transmeta.c`, `cpu/zhaoxin.c`, `cpu/hygon.c` |
| CPU vulnerability / mitigation handling | `arch/x86/kernel/cpu/bugs.c` (cross-ref dedicated `arch/x86/cpu-mitigations.md` Tier 3 to be authored) |
| CPUID dependencies + capabilities | `arch/x86/kernel/cpu/cpuid-deps.c`, `arch/x86/include/asm/cpufeatures.h`, `arch/x86/include/asm/disabled-features.h` |
| Topology (cores / threads / packages) | `arch/x86/kernel/cpu/topology.c`, `topology_amd.c`, `topology_common.c`, `topology_ext.c` |
| Microcode loading | `arch/x86/kernel/cpu/microcode/core.c`, `intel.c`, `amd.c` |
| MTRR (legacy memory-type range registers) | `arch/x86/kernel/cpu/mtrr/` |
| Cache info | `arch/x86/kernel/cpu/cacheinfo.c` |
| Early-boot setup_arch + per-CPU setup | `arch/x86/kernel/setup.c`, `setup_percpu.c`, `head64.c`, `head_64.S` |
| Process state save/restore (per-task arch context) | `arch/x86/kernel/process.c`, `process_64.c` |
| SMP CPU bringup | `arch/x86/kernel/smp.c`, `smpboot.c` |
| LAPIC / IO-APIC / x2APIC | `arch/x86/kernel/apic/` (apic.c, apic_common.c, apic_flat_64.c, apic_noop.c, apic_numachip.c, apic_physflat.c, apic_x2apic_phys.c, apic_x2apic_uv_x.c, ipi.c, msi.c, vector.c, x2apic_uv.c, …) |
| FPU save/restore | `arch/x86/kernel/fpu/` (core.c, init.c, regset.c, signal.c, xstate.c, bugs.c) |
| Time-keeping | `arch/x86/kernel/time.c`, `tsc.c`, `tsc_sync.c`, `tsc_msr.c`, `hpet.c`, `rtc.c` |
| Runtime code patching | `arch/x86/kernel/alternative.c`, `jump_label.c`, `static_call.c` |
| Kernel-module relocation | `arch/x86/kernel/module.c` |
| Paravirt | `arch/x86/kernel/paravirt.c`, `arch/x86/include/asm/paravirt*.h` |
| Intel CET (shadow stack + IBT) | `arch/x86/kernel/cet.c`, `arch/x86/kernel/shstk.c` |
| kCFI | `arch/x86/kernel/cfi.c` |
| Stack unwind / dump | `arch/x86/kernel/dumpstack.c`, `dumpstack_64.c`, `unwind_orc.c`, `unwind_frame.c`, `unwind_guess.c` |
| Early printk | `arch/x86/kernel/early_printk.c` |
| Headers | `arch/x86/include/asm/processor.h`, `desc.h`, `msr.h`, `msr-index.h`, `cpufeatures.h` |

### compatibility contract

### `/proc/cpuinfo`

Compat-critical. Format-identical to upstream. Owned by a sub-doc `arch/x86/cpuinfo.md` (Tier 3, separate); this doc owns the *gathering* of the data. Fields: vendor_id, cpu family, model, model name, stepping, microcode, cpu MHz, cache size, physical id, siblings, core id, cpu cores, apicid, initial apicid, fpu, fpu_exception, cpuid level, wp, flags, bugs, bogomips, clflush size, cache_alignment, address sizes, power management.

### MSR access

`/dev/cpu/<n>/msr` ABI: 8-byte read/write at MSR-numbered offsets. Userspace tools: `rdmsr`, `wrmsr` from `msr-tools`. Identical.

### CPUID exposure

`/dev/cpu/<n>/cpuid` ABI: 16-byte read returning EAX/EBX/ECX/EDX from CPUID at offset = (function | (subleaf << 32)). Identical.

### Architectural constants

- HWCAP / HWCAP2 bit definitions (`arch/x86/include/asm/cpufeatures.h`) — bit positions identical so `getauxval(AT_HWCAP)` returns the same bits per CPU
- `cpu_dev->c_init` per-vendor function-pointer table layout — internal but kbd / drgn / crash inspect via symbol

### Time-keeping ABI

- `clock_gettime(CLOCK_MONOTONIC)` resolution + accuracy match upstream
- TSC calibration: target same calibration accuracy
- HPET fallback when TSC is unstable
- RTC at boot read identically (`/sys/class/rtc/rtc0/`)

### Runtime patching

- `jump_label` (static branches) ABI: kernel modules can register and toggle static branches via `static_branch_enable`/`static_branch_disable`; semantics match upstream
- `static_call` (static-call ops): module-loadable virtual-call optimization; same ABI
- `alternative` (CPU-feature-gated assembly patches): match upstream alternative-table format so external tooling can introspect

### CET (Control-flow Enforcement Technology)

`arch/x86/kernel/cet.c` + `shstk.c` implement Intel CET userspace shadow-stack support. ABI:
- `arch_prctl(ARCH_SHSTK_ENABLE)`, `ARCH_SHSTK_DISABLE`, `ARCH_SHSTK_LOCK`, `ARCH_SHSTK_UNLOCK`
- `map_shadow_stack(2)` syscall
- ELF property bits in `.note.gnu.property` (NT_GNU_PROPERTY_X86_FEATURE_1_SHSTK / IBT)
- `glibc` ld.so reads the property and enables CET if requested

Identical to upstream. (Note: this is independent of Rookery's NEW exec-gain ELF note; both can coexist on the same binary.)

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| GDT/IDT/TSS install | `kani::proofs::arch::x86::desc::install_safety` |
| MSR read/write | `kani::proofs::arch::x86::msr::access_safety` (inherited from `arch/x86/00-overview.md`) |
| CPUID instruction | `kani::proofs::arch::x86::cpuid::leaf_safety` |
| FPU XSAVE/XRSTOR | `kani::proofs::arch::x86::fpu::xstate_safety` (inherited) |
| jump_label runtime patch | `kani::proofs::arch::x86::patching::jump_label_safety` |
| Module relocation | `kani::proofs::arch::x86::module::reloc_safety` |
| APIC IPI delivery | `kani::proofs::arch::x86::apic::ipi_safety` |
| Stack unwinder ORC | `kani::proofs::arch::x86::unwind::orc_safety` |

### Layer 2: TLA+ models

- `models/arch/x86/smp_boot.tla` (inherited from `arch/x86/00-overview.md` Layer-2; co-owned with `arch/x86/boot.md`) — proves SMP bringup invariants
- `models/arch/x86/microcode_update.tla` (NEW) — proves microcode update is observed atomically across all CPUs after stop_machine resumes
- `models/arch/x86/apic_vector_alloc.tla` (NEW) — proves IRQ-vector allocation is collision-free across concurrent device-attach paths

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY for `arch/x86/`)

Per `arch/x86/00-overview.md` Layer-3 mandates:

| Data structure | Invariant | Harness |
|---|---|---|
| IDT entry table | Vectors 0–31 map to CPU-fixed handler kind; 32–255 either unset or registered IRQ | `kani::proofs::arch::x86::idt::install_invariants` (inherited) |
| TSS layout | Each per-CPU TSS has IST stacks within per-CPU stack area; no overlap with kernel direct-map | `kani::proofs::arch::x86::tss::ist_layout_invariants` (inherited) |
| Per-CPU offset table | Per-CPU access never crosses per-CPU boundary | `kani::proofs::arch::x86::cpu::percpu_invariants` (inherited; Rookery to author per issue #4) |
| GDT layout | Kernel CS/DS/SS, user CS/DS/SS, TSS descriptor at fixed offsets per upstream | `kani::proofs::arch::x86::gdt::layout_invariants` |
| FPU xstate buffer | Save area size matches XSAVE feature mask; component offsets match CPUID 0xD lookup | `kani::proofs::arch::x86::fpu::xstate_layout_invariants` |

### Layer 4: Functional correctness (opt-in)

- **TSC calibration accuracy** via Verus — small spec; high-leverage (every clock_gettime depends).
- **APIC vector allocation** via Creusot — bounded data structure with clear correctness criterion.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **PRIVATE_KSTACKS** | Per-CPU TSS holds per-task `rsp0` (kernel stack); each task has a unique kernel stack | § Mandatory |
| **CLOSE_KERNEL/CLOSE_USERLAND** (SMEP+SMAP) | CR4 bits set early in `cpu_init` and never cleared | § Mandatory |
| **kCFI** | `arch/x86/kernel/cfi.c` implements forward-edge CFI; CONFIG_CFI_CLANG=y default-on | § Default-on configurable off |
| **CET shadow stack** (Intel) | `arch/x86/kernel/{cet,shstk}.c` enables IA32_S_CET + IA32_U_CET when CPU supports | § Default-on configurable off (CONFIG_X86_USER_SHADOW_STACK=y default) |
| **DIRECT_CALL** (Spectre-v2 mitigation kernel side) | `arch/x86/kernel/cpu/bugs.c` selects the per-CPU mitigation; runtime patches retpoline ↔ direct call | § Default-on configurable off |

### Row-1 features consumed by this component

- **KERNEXEC**: this component's text + rodata are RX/RO; runtime patching uses `text_poke` via cpu_idle stop-and-go protocol
- **SIZE_OVERFLOW**: TSC offset arithmetic, microcode firmware-size bounds use checked operators
- **CONSTIFY**: descriptor tables (GDT/IDT entries) are static const tables; vendor `c_init` function-pointer tables are `static const`

### Row-2 / GR-RBAC integration

This component runs partly before LSM is initialized (CPU bringup runs during early init). Post-LSM-init, MSR/CPUID device-node access (`/dev/cpu/<n>/{msr,cpuid}`) is mediated by file_permission LSM hooks, which GR-RBAC policy can enforce additionally. The existing `CAP_SYS_RAWIO` requirement is preserved.

### Userspace-visible behavior changes

None beyond the upstream-default behavior. CET shadow stack default-on matches upstream's per-CPU default. kCFI default-on matches upstream's CONFIG_CFI_CLANG=y default for clang builds.

### Verification

(See § Verification above.)

