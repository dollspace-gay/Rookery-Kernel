---
title: "Subsystem: arch/ — architecture support"
tags: ["design-doc", "subsystem"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-2 overview for the `arch/` subsystem. v0 implements only x86_64 — see `arch/x86/00-overview.md` for the substantive design. This document records the per-architecture conventions every future arch port will follow and the design-doc layout for arch tiers.

Per `00-overview.md` D3, the arch-abstraction layer (`include/asm-generic/`, `include/linux/`'s arch-agnostic interfaces) is NOT designed up front in v0. `arch/x86/` is concrete and stands alone. When a second architecture is added (v1+), the arch-abstraction layer becomes a refactoring concern at that time.

### Requirements

- REQ-1: An architecture's `00-overview.md` lists every Tier-3 doc spawned under that arch directory.
- REQ-2: An architecture's `00-overview.md` declares which userspace-visible compat-surface elements are owned by that arch (the rows of the table above).
- REQ-3: An architecture's `00-overview.md` declares its in-v0 / deferred-v1 status in the frontmatter.
- REQ-4: At least the ten arch-tier topics above (boot / entry / paging / per-CPU / atomics / signal / vDSO / kexec / virt-guest / cpu-init) are addressed in the arch's overview, even if the discussion is "delegated to Tier 3 doc <X>".

### Acceptance Criteria

- [ ] AC-1: `arch/x86/00-overview.md` exists and addresses the ten arch-tier topics. (covers REQ-4 for x86)
- [ ] AC-2: A grep for "status: deferred-v1" in `.design/arch/*/00-overview.md` finds zero matches in v0 (no deferred arch overviews exist yet); when arch ports begin in v1+, this expectation flips.
- [ ] AC-3: `arch/x86/00-overview.md` enumerates planned Tier-3 docs covering all of: boot, entry, paging, per-CPU, atomics, signal, vDSO, kexec, virt-guest, cpu-init. (covers REQ-1 for x86)
- [ ] AC-4: `arch/x86/00-overview.md` claims the userspace-visible compat-contract rows owned by x86 (the table above). (covers REQ-2 for x86)

### Architecture

### Why no upfront abstraction layer

An arch-abstraction layer is the kind of premature architecture that the upstream Linux kernel has spent decades cleaning up. `include/asm-generic/` exists precisely because retrofitting an abstraction after multiple concrete arches were written produced a more honest interface than designing one before any arch existed.

Rookery v0 commits to this lesson:
1. Write `arch/x86/` against concrete x86_64 hardware.
2. Build subsystem-level abstractions (in `mm/`, `kernel/`, `fs/`, etc.) such that they import from `arch::*` rather than directly poking at hardware. The exact shape of `arch::*` is allowed to be x86-shaped in v0.
3. When a second arch lands (v1+), drive abstraction extraction by what the second arch actually needs different. This produces an interface whose "abstraction debt" is paid by the work of porting a second arch — a known-good model.

Trade-off acknowledged: when v1+ ports begin, some `arch::*` interfaces will need refactoring. Per `00-overview.md` D5, this is the deliberate plan.

### Layout map

```
.design/arch/
  00-overview.md          ← this document
  x86/
    00-overview.md        ← Tier-2 substantive doc
    boot.md               ← Tier 3
    entry.md              ← Tier 3
    paging.md             ← Tier 3
    kernel-platform.md    ← Tier 3 (CPU bringup, IDT, TSS, FPU, MSRs, time)
    signal.md             ← Tier 3
    vdso.md               ← Tier 3
    cpuinfo.md            ← Tier 3
    ptrace-abi.md         ← Tier 3
    elf.md                ← Tier 3
    cpu-mitigations.md    ← Tier 3 (Spectre, Meltdown, MDS, retbleed, ...)
    coco.md               ← Tier 3 (SEV-SNP guest, TDX guest)
    hyperv-guest.md       ← Tier 3
    xen-guest.md          ← Tier 3
    kvm.md                ← Tier 3 (host-side; cross-references virt/00-overview)
    pci.md                ← Tier 3
    ras.md                ← Tier 3 (machine-check, RAS-CEC)
    power.md              ← Tier 3 (suspend, hibernate)
    crypto-accel.md       ← Tier 3 (AES-NI, SHA-NI, AVX assists for crypto/)
```

(Other architectures, when added, mirror this layout under `arch/<name>/`.)

### Out of Scope

- All architectures other than x86_64 in v0.
- The arch-abstraction layer itself (deferred to v1+ when a second arch lands).
- 32-bit x86 (`i386`) — even within `arch/x86/`, only the `X86_64=y` configuration is in v0 scope. The `arch/x86/ia32/` 32-bit-on-64-bit compat layer is in scope (because existing 32-bit userspace must still run); but a pure `i386` kernel is not.
- User Mode Linux (`arch/um/`) — pending Q2 confirmation.
- Architecture-specific cryptographic accelerators outside x86 (e.g. ARM Crypto Extensions) — covered when those arches are added.

### upstream references in scope

- `arch/` (top-level — 22 architecture directories at baseline)
- `arch/Kconfig` (architecture-feature `config HAVE_*` definitions)
- `include/asm-generic/` (architecture-default fallback headers — referenced for context, not redesigned in v0)

Architectures upstream supports at baseline `27a26ccfd528`:
`alpha`, `arc`, `arm`, `arm64`, `csky`, `hexagon`, `loongarch`, `m68k`, `microblaze`, `mips`, `nios2`, `openrisc`, `parisc`, `powerpc`, `riscv`, `s390`, `sh`, `sparc`, `um`, `x86`, `xtensa`.

Of these, **only `x86` is in v0 scope** (and within `x86`, only the `X86_64=y` configuration). Every other arch directory is OUT OF SCOPE for v0; placeholder design docs may be added later under `arch/<name>/00-overview.md` with `status: deferred-v1`.

### compatibility contract (arch tier)

The arch tier owns the parts of the drop-in compat surface that are visible to userspace specifically through CPU-architecture details:

| Interface | Owner doc | Compat level |
|---|---|---|
| ELF e_machine value (`EM_X86_64=62`) and ELF format flavor | `arch/x86/00-overview.md` + `arch/x86/elf.md` (Tier 3) | Identity |
| Userspace register layout in signal frames (`struct ucontext_t`, `struct sigcontext`) | `arch/x86/signal.md` (Tier 3) | Byte-identical |
| Syscall ABI: register usage, syscall instruction (`syscall`/`int 0x80`/`sysenter`), errno-return convention | `arch/x86/entry.md` (Tier 3) | Byte-identical |
| Syscall numbers (`arch/x86/entry/syscalls/syscall_64.tbl`) | `arch/x86/00-overview.md` references; per-syscall semantics in Tier-5 `uapi/syscalls/*.md` | Numeric identity |
| vDSO contents (`linux-vdso.so.1` symbols) | `arch/x86/vdso.md` (Tier 3) | Symbol + ABI identity |
| Boot protocol (`bzImage` header, `boot_params`) | `arch/x86/boot.md` (Tier 3) | Byte-identical |
| Kexec hand-off (`segment[]` + `boot_params` reconstruction) | `arch/x86/boot.md` (Tier 3) | Byte-identical (per REQ-15) |
| Page table layout (4-level / 5-level, page sizes 4 KiB / 2 MiB / 1 GiB) | `arch/x86/paging.md` (Tier 3) | Identity (otherwise userspace `mmap` and pmap output diverge) |
| Userspace virtual-address space limits (`TASK_SIZE`, `mmap` randomization range) | `arch/x86/paging.md` (Tier 3) | Identity (visible via `/proc/<pid>/maps`) |
| `/proc/cpuinfo` content format | `arch/x86/cpuinfo.md` (Tier 3) | Field identity |
| ptrace / perf register sets (`PTRACE_GETREGS`, `PTRACE_GETFPREGS`, perf user-regs ABI) | `arch/x86/ptrace-abi.md` (Tier 3) | Byte-identical |

When a second architecture is added (v1+), the arch-abstraction layer must preserve the same set of interfaces in its arch-agnostic form (`include/uapi/linux/`-defined types remain identity-preserving across arches; arch-private interfaces in `include/uapi/asm/` are arch-scoped).

### per-architecture design-doc convention

Every architecture's `arch/<name>/00-overview.md` (Tier 2) MUST declare:

1. **In-scope or deferred** — `status: in-v0` or `status: deferred-v1`.
2. **Boot path** — how control reaches the kernel from the bootloader on this arch (Tier 3 `arch/<name>/boot.md`).
3. **Entry path** — syscall, IRQ, exception entry/exit (Tier 3 `arch/<name>/entry.md`).
4. **Paging model** — page table levels, page sizes, ASID/PCID handling, TLB flushing (Tier 3 `arch/<name>/paging.md`).
5. **Per-CPU state model** — how per-CPU areas are addressed (`%gs:` on x86_64, TPIDR_EL1 on ARM64, etc.) (Tier 3 `arch/<name>/percpu.md` or folded into kernel-platform doc).
6. **Atomic primitives** — how `atomic_t` operations map to instructions, memory ordering (Tier 3 `arch/<name>/atomics.md` or folded into kernel-platform doc).
7. **Signal frame layout** — userspace-visible (Tier 3 `arch/<name>/signal.md`).
8. **vDSO** — symbols exported and how page-table-mapped (Tier 3 `arch/<name>/vdso.md`).
9. **Kexec/kdump** — if supported (Tier 3 inside `arch/<name>/boot.md`).
10. **Confidential / virt-guest support** — SEV, TDX, pKVM, Xen guest, Hyper-V, etc. (Tier 3 per-feature docs).

Plus the standard subsystem-doc sections: Compatibility contract, Requirements, Acceptance Criteria, Architecture, Verification, Hardening, Open Questions, Out of Scope.

### verification

This Tier-2 document does not directly own implementation code; verification is delegated to the Tier-3 docs it spawns. Each Tier-3 doc declares its own:
- Anticipated `unsafe` blocks and their Kani-harness names (Layer 1)
- TLA+ models for any concurrency primitive or invariant (Layer 2)
- Kani harnesses for data-structure invariants (Layer 3) — particularly relevant for `paging.md` (page-table walker invariants)
- Creusot/Verus/Prusti opt-ins (Layer 4) — none mandatory at this tier

A meta-verification check: `make verify-arch` runs every harness under `proofs/arch/**/*` and every model under `models/arch/**/*`, reporting per-arch coverage. `make verify-strict-arch` adds Layer 4.

### hardening

(Placeholder per `00-overview.md` D6. Binding hardening policy lands in `00-security-principles.md` after that document is authored. The arch tier owns the implementation of: KASLR, SMEP, SMAP, Intel CET, retpolines, IBPB, RAP-equivalent kCFI on entry/exit boundaries, PRIVATE_KSTACKS, RANDKSTACK. Each Tier-3 arch/x86/ doc that touches these will gain a "Hardening" section once the principles doc lands.)

