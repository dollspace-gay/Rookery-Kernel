---
title: "Tier-3: arch/x86/vdso — vDSO + vsyscall page"
tags: ["design-doc", "tier-3", "arch-x86"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the x86_64 vDSO (virtual Dynamic Shared Object) and the legacy vsyscall page. The vDSO is a small kernel-provided shared object mapped into every userspace process; it exports fast-path syscalls (`__vdso_clock_gettime`, `__vdso_gettimeofday`, `__vdso_getcpu`, `__vdso_time`) that execute entirely in userspace without entering the kernel. The vsyscall page is a legacy fixed mapping at `0xffffffffff600000` that EMULATES three legacy syscalls for very old binaries. Owns the vDSO ELF-image build, the per-process mapping setup at exec time, the seqlock-coordinated clock-source data shared between kernel writers and userspace readers, and the signal restorer (`__kernel_rt_sigreturn`) consumed by `arch/x86/signal.md`.

### Requirements

- REQ-1: vDSO ELF image bit-identical to upstream's vDSO ELF image — same dynamic-symbol table, same entry points, same .text content (modulo a few absolute addresses that the kernel patches in at boot for the vdso_data page address).
- REQ-2: Each vDSO function (`__vdso_clock_gettime`, etc.) returns identical bytes to userspace for identical inputs (clock_id, timestamp).
- REQ-3: vDSO is mapped per-process at exec time at a randomized address (`vvar`/`vdso` regions). `AT_SYSINFO_EHDR` aux entry points to the vDSO base.
- REQ-4: `vdso_data` struct layout byte-identical to upstream; updates use seqlock; readers retry on `seq & 1`.
- REQ-5: vsyscall page in EMULATE mode by default per `arch/x86/00-overview.md` D5: page mapped at `0xffffffffff600000`; on userspace call, #PF; emulator translates to the corresponding syscall.
- REQ-6: 32-bit vDSO (for ia32 compat layer): `__kernel_vsyscall`, `__kernel_sigreturn`, `__kernel_rt_sigreturn`, plus 32-bit aliases of the time-related symbols.
- REQ-7: vDSO + `arch/x86/signal.md` integration: signal restorer set to vDSO-exported `__kernel_rt_sigreturn` so the standard signal-return path goes through the vDSO trampoline.
- REQ-8: Layer-2 TLA+ model `models/lib/vdso_seqlock.tla` (cross-ref `lib/00-overview.md` Layer 2) proves writer-reader coordination — the writer side is owned here; the reader side runs in userspace.
- REQ-9: SGX enclave entry (`__vdso_sgx_enter_enclave`) preserved when CONFIG_X86_SGX=y.
- REQ-10: vDSO build integrates with kbuild per upstream's `arch/x86/entry/vdso/Makefile`; produces both 32-bit and 64-bit ELF images.
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `objdump -d arch/x86/entry/vdso/vdso64/vdso.so.dbg` byte-identical content vs. upstream's vdso.so.dbg (modulo absolute addresses that the kernel relocates at boot). (covers REQ-1)
- [ ] AC-2: A `clock_gettime(CLOCK_MONOTONIC, &ts)` round-trip via vDSO returns the same bytes on Rookery vs. upstream (modulo time-of-call). Statistical: 1000 calls fall within 1µs of each other across the two kernels. (covers REQ-2)
- [ ] AC-3: `getauxval(AT_SYSINFO_EHDR)` returns the vDSO base; `dlsym(handle, "__vdso_clock_gettime")` returns a function pointer that, when called, behaves correctly. (covers REQ-3)
- [ ] AC-4: `pahole struct vdso_data` byte-identical layout vs. upstream. (covers REQ-4)
- [ ] AC-5: A test calling the legacy vsyscall page (constructable via `((time_t (*)(time_t *)) 0xffffffffff600400)(NULL)`) returns the current time on Rookery in EMULATE mode. (covers REQ-5)
- [ ] AC-6: A 32-bit-on-64-bit `gettimeofday` userspace test via the 32-bit vDSO returns the same time. (covers REQ-6)
- [ ] AC-7: A signal handler test verifies that the signal restorer is in the vDSO range; sigreturn dispatches correctly. (covers REQ-7)
- [ ] AC-8: `make tla` passes `models/lib/vdso_seqlock.tla`. (covers REQ-8)
- [ ] AC-9: An SGX-enabled kernel + an SGX-using userspace test (e.g., `tools/testing/selftests/sgx/`) passes. (covers REQ-9)
- [ ] AC-10: `make` produces `arch/x86/entry/vdso/vdso64.so` + `arch/x86/entry/vdso/vdso32.so`; readelf-comparison against upstream is identical. (covers REQ-10)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

The vDSO is a userspace ELF image; the kernel side is the build orchestration + the per-process mapping + the `vdso_data` writer.

- `kernel::arch::x86::vdso::vma` — per-process vDSO + vvar VMA mapping at exec
- `kernel::arch::x86::vdso::data_writer` — `vdso_data` seqlock writer side
- `kernel::arch::x86::vdso::vsyscall_emulator` — vsyscall page emulator
- `kernel::arch::x86::vdso::sgx_enter` — SGX enclave entry helper

The userspace-side vDSO `.text` is C + assembly written outside the Rust kernel (it runs in userspace, not the kernel; same as upstream); built via kbuild per upstream's `arch/x86/entry/vdso/Makefile`.

### Locking and concurrency

- **`vdso_data` writer side** (in this component): seqlock — increment `seq` (now odd), write fields, increment `seq` (now even). Single writer (the timekeeping update path).
- **Reader side** (in userspace, exported by the vDSO): seqcount-based retry — read `seq`, if odd retry; read fields; check `seq` unchanged; if changed, retry.

TLA+ model `models/lib/vdso_seqlock.tla` (cross-ref `lib/00-overview.md`) proves: every reader either observes a coherent snapshot or retries.

### Error handling

vDSO functions return per-syscall conventions: 0 on success, negative errno on failure. The kernel side never fails to map the vDSO at exec time except in OOM (returns -ENOMEM from execve).

vsyscall emulation faults: if the user-side call signature is bad, deliver SIGSEGV to the calling task.

### Out of Scope

- Other-arch vDSO (cross-ref `lib/00-overview.md` § vdso-core.md for cross-arch infrastructure)
- vsyscall NONE / XONLY default (out per D5; EMULATE is the default)
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| x86 vDSO image build | `arch/x86/entry/vdso/` |
| Per-process vDSO mapping | `arch/x86/entry/vdso/vma.c` |
| 32-bit vDSO build + setup | `arch/x86/entry/vdso/vdso32-setup.c`, `arch/x86/entry/vdso/vdso32/` |
| 64-bit vDSO image | `arch/x86/entry/vdso/vdso64/` |
| Legacy vsyscall page | `arch/x86/entry/vsyscall/` |
| Cross-arch vDSO core | `lib/vdso/` (cross-ref `lib/00-overview.md` § vdso-core.md) |
| Public vdso headers | `include/vdso/` |

### compatibility contract

### vDSO symbols (64-bit)

`linux-vdso.so.1` exports byte-for-byte identical symbols vs. upstream:

| Symbol | C signature |
|---|---|
| `__vdso_clock_gettime` | `int __vdso_clock_gettime(clockid_t clk_id, struct timespec *tp)` |
| `__vdso_clock_getres` | `int __vdso_clock_getres(clockid_t clk_id, struct timespec *tp)` |
| `__vdso_gettimeofday` | `int __vdso_gettimeofday(struct timeval *tv, struct timezone *tz)` |
| `__vdso_time` | `time_t __vdso_time(time_t *t)` |
| `__vdso_getcpu` | `int __vdso_getcpu(unsigned *cpu, unsigned *node, void *unused)` |
| `__vdso_sgx_enter_enclave` | (Intel SGX entry; CONFIG_X86_SGX) |
| Aliases: `clock_gettime`, `clock_getres`, `gettimeofday`, `time`, `getcpu` (POSIX-named, undecorated; declared via `__VDSO_USAGE` mechanism so userspace can find by either name) | (same signatures) |
| `__kernel_rt_sigreturn`, `__kernel_sigreturn` (for legacy 32-bit signals) | signal restorer (cross-ref `arch/x86/signal.md`) |

ELF dynamic symbol table content byte-identical so `dlsym` / `getauxval(AT_SYSINFO_EHDR)` lookups return the same addresses (modulo KASLR-randomized ASLR).

### `vdso_data` shared structure

The kernel maintains a `vdso_data` struct (in `lib/vdso/`) shared READ-ONLY-from-userspace via the vDSO data page. Layout byte-identical to upstream so vDSO-using userspace works unmodified. Fields:

- `seq` (seqcount for reader-side retry)
- `clock_mode` (current clocksource type: TSC, HPET, paravirt, no-clock)
- `cycle_last` (latest TSC sample)
- `mask` (TSC mask)
- `mult`, `shift` (cycle → ns conversion)
- `clock_id_to_ns` (per-clock-id offset table)
- `tz_minuteswest`, `tz_dsttime` (timezone info for gettimeofday)
- `wall_time_sec`, `wall_time_nsec` (wall clock)
- `monotonic_time_sec`, `monotonic_time_nsec`
- `boot_time_sec`, `boot_time_nsec`
- Per-CPU info for getcpu: `cpu`, `node`

Updates are seqlock-coordinated; userspace re-reads on retry. Cross-ref `lib/00-overview.md` § vdso-core.md for the cross-arch generic logic.

### vsyscall page (legacy)

Fixed mapping at virtual address `0xffffffffff600000`, exposing three pages of stubs. Upstream supports three modes via CONFIG_LEGACY_VSYSCALL_*:
- **EMULATE** (default; matches Rookery per `arch/x86/00-overview.md` D5): kernel intercepts user code at the vsyscall address (#PF) and emulates the syscall
- **XONLY**: page is mapped X-only; calls work but reads of the page fault (defends against ROP gadget extraction)
- **NONE**: page is unmapped; calls fault

Rookery defaults to EMULATE per D5.

### Signal restorer

`__kernel_rt_sigreturn` (and `__kernel_sigreturn` for legacy 32-bit signals) — exported by the vDSO. Used by `arch/x86/signal.md` as the user-mode trampoline that returns from the user signal handler back to the kernel via `rt_sigreturn` syscall.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| vDSO + vvar VMA mapping at exec | `kani::proofs::arch::x86::vdso::map_safety` |
| `vdso_data` seqlock writer | `kani::proofs::arch::x86::vdso::data_writer_safety` |
| vsyscall emulator pt_regs read | `kani::proofs::arch::x86::vdso::vsyscall_emulator_safety` |

### Layer 2: TLA+ models

- `models/lib/vdso_seqlock.tla` (mandatory per `lib/00-overview.md` Layer 2; co-owned here for the writer side)

### Layer 3: Kani harnesses for data-structure invariants

- `vdso_data` layout invariants — every field at the documented offset; checked at compile time via const assertions

### Layer 4: Functional correctness (opt-in)

- **vDSO time conversion** (cycle → nanoseconds via `mult` + `shift`) via Verus — proves no overflow over 600 years. Already part of upstream's TSC math; Rookery proves correctness.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **vsyscall XONLY mode** (defense-in-depth, NOT default) | Page mapped X-only; defeats ROP gadget extraction. Per `arch/x86/00-overview.md` D5 we default EMULATE; XONLY remains an opt-in via `vsyscall=xonly` boot param. | (cross-ref `arch/x86/00-overview.md`) |
| **vDSO mapping randomization** | Per-process random offset within the vDSO/vvar mapping range; matches upstream `arch_vma_name`/`arch_get_unmapped_area` randomization | § Default-on configurable off (matches upstream) |

### Row-1 features consumed by this component

- **KERNEXEC**: vDSO ELF text is RX in userspace; vvar is RO+NX in userspace
- **UDEREF**: vsyscall emulator reads pt_regs; never raw user-VA deref
- **REFCOUNT**: vDSO mapping refcount is `Refcount`
- **CONSTIFY**: vDSO ELF image is `static const` data baked into the kernel image

### Row-2 / GR-RBAC integration

vDSO mapping happens in exec; LSM hooks fire from `fs/exec-binfmt.md`. This component does not call LSM hooks directly.

### Userspace-visible behavior changes

None beyond upstream defaults. The vDSO + vsyscall provide identical contracts.

### Verification

(See § Verification above.)

