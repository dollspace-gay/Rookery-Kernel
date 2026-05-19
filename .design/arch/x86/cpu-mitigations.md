# Tier-3: arch/x86/cpu-mitigations — Spectre, Meltdown, MDS, retbleed, kCFI, CET, IBT, retpolines

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - arch/x86/kernel/cpu/bugs.c
  - arch/x86/include/asm/nospec-branch.h
  - arch/x86/lib/retpoline.S
  - arch/x86/kernel/callthunks.c
  - arch/x86/kernel/cfi.c
  - arch/x86/kernel/cet.c
  - arch/x86/kernel/shstk.c
  - arch/x86/kernel/ibt_selftest.S
  - arch/x86/include/asm/ibt.h
  - arch/x86/include/asm/cfi.h
  - include/linux/cfi_types.h
  - include/linux/cfi.h
  - arch/x86/mm/pti.c
  - Documentation/admin-guide/hw-vuln/
  - Documentation/admin-guide/kernel-parameters.txt
-->

## Summary
Tier-3 design for x86_64 CPU-vulnerability mitigations and control-flow integrity. Owns the suite of speculative-execution mitigations (Spectre v1/v2, Meltdown via KPTI, MDS, TAA, SRBDS, MMIO Stale Data, retbleed, GDS, BHI, SLS, EntryBleed, RFDS, Reg-File Data Sampling), kCFI (clang-CFI for indirect calls), Intel CET shadow stack + IBT, retpoline thunks, IBPB / IBRS / eIBRS / LASS infrastructure, callthunks (kCFI-compatible call thunking), and the `/sys/devices/system/cpu/vulnerabilities/*` reporting tree.

This Tier-3 owns the implementation of the row-1 hardening features **DIRECT_CALL** (Spectre-v2 retpoline + IBRS), **RAP/CFI** (kCFI), and **CLOSE_KERNEL/CLOSE_USERLAND** (SMEP/SMAP) as specified in `00-security-principles.md`'s Locked-in row-1 absorption list.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Per-CPU vulnerability detection + mitigation selection | `arch/x86/kernel/cpu/bugs.c` |
| Speculative-mitigation primitives | `arch/x86/include/asm/nospec-branch.h`, `arch/x86/lib/retpoline.S` |
| Call thunks (kCFI-compatible) | `arch/x86/kernel/callthunks.c` |
| kCFI | `arch/x86/kernel/cfi.c`, `arch/x86/include/asm/cfi_types.h` |
| Intel CET (shadow stack + IBT) | `arch/x86/kernel/cet.c`, `arch/x86/kernel/shstk.c` |
| IBT helpers | `arch/x86/kernel/ibt_selftest.S`, `arch/x86/include/asm/ibt.h` |
| KPTI (Meltdown mitigation) | `arch/x86/mm/pti.c` (cross-ref `arch/x86/paging.md`) |
| Userspace-visible documentation | `Documentation/admin-guide/hw-vuln/` (per-vulnerability docs) |
| Kernel command-line knobs | `Documentation/admin-guide/kernel-parameters.txt` (`spectre_v2=`, `mds=`, `tsx_async_abort=`, `srbds=`, `mmio_stale_data=`, `retbleed=`, `gather_data_sampling=`, `spectre_bhi=`, `spectre_v2_user=`, `mitigations=`, etc.) |

## Compatibility contract

### `/sys/devices/system/cpu/vulnerabilities/*`

Per-vulnerability sysfs files reporting "Mitigation: <strategy>" or "Not affected" or "Vulnerable: ..." per upstream's `cpu_show_*()` functions. **Format-identical** to upstream so tools (`spectre-meltdown-checker`, distro security audits, lscpu) parse identically.

Files at baseline:
- `gather_data_sampling`
- `itlb_multihit`
- `l1tf`
- `mds`
- `meltdown`
- `mmio_stale_data`
- `retbleed`
- `spec_rstack_overflow`
- `spec_store_bypass`
- `spectre_v1`
- `spectre_v2`
- `srbds`
- `tsx_async_abort`
- `bhi` (Branch History Injection)
- `entrybleed`
- `reg_file_data_sampling`

### Kernel command-line knobs

Each vulnerability has a per-vuln knob (e.g., `spectre_v2=on`, `spectre_v2=off`, `spectre_v2=auto`, `spectre_v2=ibrs`, `spectre_v2=eibrs`, …) plus a master `mitigations=auto|on|off|auto,nosmt`. Identical to upstream syntax + semantics.

### kCFI signature scheme

kCFI inserts a 32-bit type-signature literal before each function. Indirect calls verify the literal matches the expected function-type's signature. The signature scheme matches upstream's clang-CFI implementation; signatures must be byte-identical so out-of-tree modules built against upstream load on Rookery (recompile-from-source compat per `00-overview.md` REQ-4 / D2).

### Intel CET userspace ABI

- `arch_prctl(ARCH_SHSTK_ENABLE)`, `ARCH_SHSTK_DISABLE`, `ARCH_SHSTK_LOCK`, `ARCH_SHSTK_UNLOCK`
- `map_shadow_stack(2)` syscall
- ELF property bits in `.note.gnu.property`: `NT_GNU_PROPERTY_X86_FEATURE_1_SHSTK`, `_IBT`
- glibc's `ld.so` reads the property and enables CET if requested

Identical to upstream.

## Requirements

- REQ-1: Every CPU-vulnerability mitigation upstream supports at the baseline commit is implemented in Rookery.
- REQ-2: Per-CPU mitigation selection (`identify_cpu` → `select_*_mitigation`) follows upstream's per-microarchitecture algorithm; same CPUs select same mitigations.
- REQ-3: `/sys/devices/system/cpu/vulnerabilities/*` content is byte-identical to upstream for the same hardware.
- REQ-4: Kernel command-line knobs preserve syntax + behavior identically; `mitigations=off` opt-out works.
- REQ-5: Retpoline thunks (`__x86_indirect_thunk_*` in `arch/x86/lib/retpoline.S`) are emitted by the build per CONFIG_RETPOLINE; runtime patching swaps retpoline ↔ direct-call when `enhanced IBRS` (eIBRS) is selected and supported.
- REQ-6: IBPB / IBRS / eIBRS MSR write sequences match upstream — at every privilege boundary that requires them.
- REQ-7: KPTI (Meltdown mitigation) — auto-enabled on Meltdown-affected CPUs (Intel pre-Whiskey-Lake); per-CPU separate user PGD; CR3 swap on entry/exit. Cross-ref `arch/x86/paging.md` for the page-table side; this Tier-3 owns the per-CPU CR3-swap orchestration.
- REQ-8: kCFI — every indirect call validates the target function's type signature; failures result in `BUG_ON` with upstream-identical formatting. CONFIG_CFI_CLANG=y default per `00-security-principles.md`.
- REQ-9: Intel CET shadow stack — when CPU supports + glibc requests via ELF property, the kernel allocates the shadow stack region, maps it as `Shadow_Stack=1` in PTE, sets IA32_S_CET / IA32_U_CET MSRs. Default-on per `00-security-principles.md`.
- REQ-10: Intel IBT (Indirect Branch Tracking) — kernel + userspace ENDBR-validated indirect branches; kernel-side IBT default-on per `arch/x86/00-overview.md` D3 / `00-security-principles.md`.
- REQ-11: SMEP + SMAP — set in CR4 by `arch/x86/kernel-platform.md` early-init; this Tier-3 doc verifies they remain set across CPU hotplug + S3 wakeup. (CLOSE_KERNEL/CLOSE_USERLAND mandate.)
- REQ-12: LASS (Linear Address Space Separation) — when CPU supports, set per upstream defaults; userspace/kernel address ranges enforced.
- REQ-13: Per-microcode-version vulnerability re-scan — when microcode updates, re-evaluate vulnerability state + emit corrected `/sys/.../vulnerabilities/*` strings.
- REQ-14: SLS (Straight-Line Speculation) mitigation — kernel-side gcc/clang `-mharden-sls=all` flag for kernel build per upstream defaults.
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: For each vulnerability file in `/sys/devices/system/cpu/vulnerabilities/`, `diff` between Rookery and upstream on identical hardware is empty. (covers REQ-1, REQ-2, REQ-3)
- [ ] AC-2: Boot with `mitigations=off`; `/sys/.../vulnerabilities/*` reports `Vulnerable` for affected CPUs. Boot with `mitigations=auto` (default); reports `Mitigation: ...` per upstream. (covers REQ-4)
- [ ] AC-3: A kernel built with CONFIG_RETPOLINE=y has `__x86_indirect_thunk_*` symbols. On an eIBRS-capable CPU, runtime patching converts them to direct calls (verifiable via `objdump -d vmlinux` + post-boot `/proc/kallsyms` cross-ref). (covers REQ-5)
- [ ] AC-4: An IBPB-instrumented kernel (CONFIG_RETPOLINE=y, no eIBRS) issues IBPB MSR write at context-switch transitions; verifiable via perf counter + bpftrace. (covers REQ-6)
- [ ] AC-5: A KPTI test on a Meltdown-affected CPU: user PGD lacks kernel direct-map entries (visible via /sys/kernel/debug/x86/dump_pagetables vs. user-side mapping); CR3 swap fires on entry/exit. (covers REQ-7)
- [ ] AC-6: A kernel built with CONFIG_CFI_CLANG=y traps an indirect-call type-mismatch test (`tools/testing/selftests/cfi/`); the trap message matches upstream. (covers REQ-8)
- [ ] AC-7: On a CET-capable CPU, an `arch_prctl(ARCH_SHSTK_ENABLE)`'d userspace process gets a shadow stack mapped; the OS x86_user_shstk selftest passes. (covers REQ-9)
- [ ] AC-8: On an IBT-capable CPU, `arch/x86/kernel/ibt_selftest.S` passes (kernel boots without spurious #CP); userspace-IBT-tagged binaries (built with `-fcf-protection=full`) execute correctly. (covers REQ-10)
- [ ] AC-9: A test reads CR4 from boot through CPU hotplug + S3 suspend/resume; SMEP+SMAP bits remain set. (covers REQ-11)
- [ ] AC-10: On a LASS-capable CPU (future hardware), LASS bits set in CR4 per upstream policy. (covers REQ-12)
- [ ] AC-11: After a microcode update via `/dev/cpu/<n>/microcode`, `/sys/.../vulnerabilities/*` strings re-scan and update. (covers REQ-13)
- [ ] AC-12: A kernel built with `-mharden-sls=all` is detected (objdump for INT3 padding after RET); `cat /sys/.../vulnerabilities/spec_rstack_overflow` reports the SLS-protected mitigation. (covers REQ-14)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::arch::x86::mitigations::detect` — per-CPU vulnerability detection (`identify_cpu` analog)
- `kernel::arch::x86::mitigations::select` — mitigation strategy selection per CPU
- `kernel::arch::x86::mitigations::apply` — apply selected mitigation (set MSRs, runtime-patch retpoline ↔ direct call, etc.)
- `kernel::arch::x86::mitigations::sysfs` — `/sys/devices/system/cpu/vulnerabilities/*` interface
- `kernel::arch::x86::mitigations::spectre_v1` — barriers (`barrier_nospec()`)
- `kernel::arch::x86::mitigations::spectre_v2` — retpoline / IBRS / eIBRS / IBPB
- `kernel::arch::x86::mitigations::meltdown::pti` — KPTI per-CPU CR3-swap orchestration
- `kernel::arch::x86::mitigations::mds` — MDS clear-CPU-buffers logic
- `kernel::arch::x86::mitigations::retbleed` — retbleed-specific (call-depth tracking, IBPB-on-VMX-exit)
- `kernel::arch::x86::mitigations::gds` — GDS (gather data sampling)
- `kernel::arch::x86::mitigations::bhi` — Branch History Injection (BHB barrier)
- `kernel::arch::x86::mitigations::callthunks` — kCFI-compatible call thunks
- `kernel::arch::x86::mitigations::cfi` — clang-CFI signature scheme
- `kernel::arch::x86::mitigations::cet` — Intel CET shadow stack
- `kernel::arch::x86::mitigations::ibt` — Intel IBT
- `kernel::arch::x86::mitigations::sls` — straight-line speculation

### Locking and concurrency

- Mitigation selection runs single-CPU at boot; no locks.
- Runtime patching (retpoline ↔ direct call swap) uses `text_poke_*` (cross-ref `arch/x86/kernel-platform.md`); stop_machine to halt all CPUs during patch.
- IBPB / IBRS MSR writes are per-CPU; no cross-CPU coordination required.
- KPTI CR3 swap is per-CPU (per-CPU `cpu_tlbstate.user_cr3`).

### Error handling

`Result<T, kernel::error::Error>` for fallible ops (mostly only at init time; runtime mitigation execution is non-failable):
- `Err(EINVAL)` — bad command-line knob value
- `Err(ENODEV)` — required CPU feature missing for selected mitigation

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| MSR write for IBRS / eIBRS | `kani::proofs::arch::x86::mitigations::ibrs_safety` |
| IBPB barrier instruction sequence | `kani::proofs::arch::x86::mitigations::ibpb_safety` |
| Retpoline thunk linkage | `kani::proofs::arch::x86::mitigations::retpoline_safety` |
| KPTI per-CPU CR3 swap | `kani::proofs::arch::x86::mitigations::pti_swap_safety` |
| kCFI signature compare | `kani::proofs::arch::x86::mitigations::kcfi_compare_safety` |
| Shadow stack alloc + mmap | `kani::proofs::arch::x86::mitigations::shstk_alloc_safety` |

### Layer 2: TLA+ models

- `models/arch/x86/ibrs_context_switch.tla` (NEW) — proves IBRS state is correctly cleared/restored across user→kernel→user transitions; no IBRS-leaked branch predictor between two userspace tasks.
- `models/arch/x86/pti_swap.tla` (inherited from `arch/x86/paging.md`) — co-owned. Proves user-PGD swap is correctly ordered with respect to entry/exit + IRET.
- `models/arch/x86/kcfi_failure_path.tla` (NEW) — proves kCFI failure (signature mismatch) results in BUG_ON, not silent-corruption / control-flow-hijack.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| `cpu_caps` array (per-CPU feature bits) | All bits set match CPUID; mitigation-required bits (CONFIG_*) set when needed | `kani::proofs::arch::x86::mitigations::cpu_caps_invariants` |
| Mitigation selection table | Per-microarchitecture entry maps to a canonical mitigation; no two entries select contradictory mitigations | `kani::proofs::arch::x86::mitigations::table_invariants` |
| Shadow stack region table | Each task with shadow stack has exactly one shadow stack mapping; address never reused while task is alive | `kani::proofs::arch::x86::mitigations::shstk_table_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `arch/x86/00-overview.md` Layer-4 list)

- **Retpoline thunk equivalence** — proves: for any indirect call target T, the retpoline thunk `__x86_indirect_thunk_*` produces the same architectural state (next RIP = T, registers preserved) as a direct `call T`, modulo the speculative-execution non-observability. Hand-written assembly proof; declared in `arch/x86/00-overview.md`.
- **kCFI signature scheme correctness** via Verus — proves: for any pair of function types (T1, T2), `cfi_signature(T1) == cfi_signature(T2)` iff T1 and T2 have the same shape (return type + arg types). Tractable; high-leverage.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **DIRECT_CALL** (Spectre-v2 / SLS mitigation) | Retpoline thunks; IBRS/eIBRS MSR writes; IBPB; runtime patching retpoline↔direct-call when eIBRS available | § Default-on configurable off (kernel cmdline `mitigations=off` opt-out preserved) |
| **RAP/CFI** | kCFI signature checks on every indirect call; CONFIG_CFI_CLANG=y default | § Default-on configurable off |
| **PRIVATE_KSTACKS** (mitigation-side support) | Per-CPU IST stacks + shadow stacks coexist; CET shadow stack default-on per CPU support | § Default-on configurable off (CONFIG_X86_USER_SHADOW_STACK=y default) |
| **CLOSE_KERNEL / CLOSE_USERLAND** | SMEP+SMAP CR4 bits set + verified across hotplug/suspend (cross-ref `arch/x86/kernel-platform.md` for early-init); LASS when supported | § Mandatory |
| **KPTI** (Meltdown mitigation) | Per-CPU user PGD; CR3 swap on entry/exit | Auto-on per CPU; matches upstream behavior |
| **CET shadow stack + IBT** | Userspace shadow stack via ELF property + arch_prctl; kernel-side IBT-tagged code | § Default-on configurable off |

### Row-1 features consumed by this component

- **KERNEXEC**: this component's text + retpoline thunks are RX/RO; runtime patching uses `text_poke` via stop_machine
- **REFCOUNT**: shadow stack region refcount uses `Refcount`
- **AUTOSLAB**: shadow stack metadata structs allocated via per-type `KmemCache`
- **SIZE_OVERFLOW**: shadow stack alignment + size arithmetic uses checked operators
- **CONSTIFY**: vulnerability-detection table is `static const`

### Row-2 / GR-RBAC integration

This component runs largely below LSM-hook layer. The userspace-CET enable/disable via `arch_prctl` IS LSM-relevant (`security_task_prctl`); GR-RBAC policy can deny.

### Userspace-visible behavior changes

None beyond upstream defaults. CET default-on matches upstream; kCFI default-on matches upstream's CONFIG_CFI_CLANG=y for clang builds.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_KERNEXEC** — W^X enforcement for kernel-text mappings; retpoline thunks, callthunks, and IBT prologues are all RX/RO with no runtime self-modification outside `text_poke` under stop_machine.
- **PAX_RAP / kCFI** — clang-CFI indirect-call signature enforcement on every indirect call (Spectre-v2 + control-flow integrity in one); per-function 32-bit type-signature literals match upstream byte-for-byte so out-of-tree modules stay compatible.
- **Intel CET shadow stack + IBT** — userspace shadow stack via `arch_prctl(ARCH_SHSTK_ENABLE)` and `map_shadow_stack(2)`; kernel-side IBT-tagged code with ENDBR-validated indirect branches.
- **PAX_USERCOPY** — bounded copy_to/from_user on `/sys/devices/system/cpu/vulnerabilities/*` sysfs paths.
- **PAX_REFCOUNT** — saturating refcount on shadow-stack region structs; never overflows on rapid arch_prctl cycles.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-task shadow-stack regions on `exit_thread`.
- **PAX_UDEREF (SMEP/SMAP/LASS)** — SMEP+SMAP CR4 bits verified-set across CPU hotplug + S3 wakeup; LASS programmed when supported.
- **PAX_RANDKSTACK** — entry path applies per-syscall stack-offset randomization (cross-ref `entry.md`).
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding in /proc and dmesg; mitigation strings carry no kernel-internal addresses.
- **GRKERNSEC_DMESG** — restrict syslog output to CAP_SYSLOG (including kCFI BUG_ON traces).
- **MITIGATION_RETPOLINE + MITIGATION_SLS** — retpoline thunks for indirect calls; INT3 padding after RET under straight-line-speculation defense.
- **KPTI per-CPU CR3 swap** — Meltdown mitigation enforced via per-CPU `user_pcid_flush_mask`; defense against L1TF / Meltdown class.
- **IBPB on cross-privilege context switch** — defense against indirect-branch-predictor poisoning.
- **eIBRS / AutoIBRS / SBPB** — enhanced/auto IBRS programmed via MSR_IA32_SPEC_CTRL + MSR_EFER._EFER_AUTOIBRS where supported.
- **Microcode-version vulnerability re-scan** — `/sys/.../vulnerabilities/*` re-evaluated after late ucode load; defense against silent capability disappearance.

Per-doc rationale: cpu-mitigations IS the kernel's primary speculative-execution defense surface; grsec/PaX hardening here layers PAX_KERNEXEC/PAX_RAP/CET/IBT/retpoline/IBPB/eIBRS/KPTI/SMAP/SMEP/LASS as overlapping defenses so a single category compromise does not collapse the whole hardening tree, and so the upstream `/sys/.../vulnerabilities` reporting remains accurate after late microcode updates.

## Open Questions

(none — CPU-mitigation strategy is fully specified by upstream Linux + microarchitecture vendor recommendations)

## Out of Scope

- 32-bit-only mitigations (CONFIG_MITIGATION_32_BIT_*) — out per `arch/x86/00-overview.md` D1
- Userspace mitigation libraries (e.g., glibc cookie randomization) — out
- Implementation code
