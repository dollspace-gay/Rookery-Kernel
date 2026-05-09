---
title: "Subsystem: security/ — security framework + LSMs"
tags: ["design-doc", "subsystem"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---

# Subsystem: security/ — security framework + LSMs

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - security/
  - security/selinux/
  - security/apparmor/
  - security/smack/
  - security/tomoyo/
  - security/yama/
  - security/loadpin/
  - security/landlock/
  - security/lockdown/
  - security/ipe/
  - security/integrity/
  - security/keys/
  - security/safesetid/
  - security/bpf/
  - include/linux/security.h
  - include/linux/lsm_hooks.h
  - include/linux/cred.h
  - include/uapi/linux/landlock.h
  - include/uapi/linux/lsm.h
-->

## Summary
Tier-2 overview for `security/` — the Linux Security Module (LSM) framework plus the in-tree LSMs. Owns the LSM hook dispatcher (`security_*` functions called pervasively from the rest of the kernel), credentials infrastructure (`commoncap`, capability checks), nine in-tree LSMs (SELinux, AppArmor, SMACK, TOMOYO, Yama, LoadPin, Landlock, Lockdown, IPE), the integrity subsystem (IMA + EVM + dm-verity-related), the kernel keyring infrastructure, the safesetid LSM, and BPF-LSM (LSM hooks attached via BPF programs).

## Upstream references in scope

| Category | Upstream paths | Planned Tier-3 doc |
|---|---|---|
| LSM framework + hook dispatcher | `security/security.c`, `security/lsm.h`, `security/lsm_init.c`, `security/lsm_audit.c`, `security/lsm_notifier.c`, `security/lsm_syscalls.c`, `security/inode.c`, `security/min_addr.c`, `include/linux/security.h`, `include/linux/lsm_hooks.h` | `lsm-framework.md` |
| Common capability checks | `security/commoncap.c`, `security/commoncap_test.c`, `include/linux/cred.h` | `capabilities.md` |
| Device cgroup (legacy v1 device-control LSM) | `security/device_cgroup.c` | folded into `kernel/cgroup/00-overview.md` cross-ref |
| SELinux | `security/selinux/` | `selinux.md` |
| AppArmor | `security/apparmor/` | `apparmor.md` |
| SMACK | `security/smack/` | `smack.md` |
| TOMOYO | `security/tomoyo/` | `tomoyo.md` |
| Yama (ptrace restriction) | `security/yama/` | `yama.md` |
| LoadPin (kernel-image origin verification) | `security/loadpin/` | `loadpin.md` |
| Landlock (unprivileged sandboxing) | `security/landlock/`, `include/uapi/linux/landlock.h` | `landlock.md` |
| Lockdown (kernel-tampering prevention) | `security/lockdown/` | `lockdown.md` |
| IPE (Integrity Policy Enforcement) | `security/ipe/` | `ipe.md` |
| Integrity (IMA + EVM) | `security/integrity/`, `security/integrity/ima/`, `security/integrity/evm/` | `integrity.md` |
| Keys (kernel keyring) | `security/keys/`, `include/linux/key.h`, `include/uapi/linux/keyctl.h` | `keys.md` |
| safesetid (setid restriction) | `security/safesetid/` | `safesetid.md` |
| BPF-LSM (LSM hooks attached via BPF) | `security/bpf/`, cross-ref `kernel/bpf/lsm.md` | `bpf-lsm.md` |
| Kconfig.hardening (compile-time hardening defaults) | `security/Kconfig.hardening` | folded into `00-security-principles.md` (Tier 1 deferred doc, issue #2) |

## Compatibility contract

### Stackable LSM dispatch

All LSMs are stackable via the upstream stackable-LSM framework. Compat: existing LSM ordering, the `lsm=` boot parameter, the `lsm.skip_cred_modify_init=` parameter, and the per-LSM Kconfig `CONFIG_LSM` defaults preserved.

### Per-LSM userspace ABI

Each LSM has its own userspace interface. Drop-in compat means each is preserved:
- **SELinux**: `/sys/fs/selinux/*` filesystem, security xattrs, AVC log format, `setenforce`/`getenforce`, `restorecon`/`setfiles`/`load_policy` userspace tools.
- **AppArmor**: `/sys/kernel/security/apparmor/*`, `apparmor_parser`, profile load semantics.
- **SMACK**: `/sys/fs/smackfs/*`.
- **TOMOYO**: `/sys/kernel/security/tomoyo/*`.
- **Yama**: `/proc/sys/kernel/yama/{ptrace_scope}`.
- **LoadPin**: `/sys/kernel/security/loadpin/*`.
- **Landlock**: `landlock_create_ruleset(2)`, `landlock_add_rule(2)`, `landlock_restrict_self(2)` — full syscall ABI.
- **Lockdown**: `/sys/kernel/security/lockdown` for current state.
- **Integrity (IMA + EVM)**: `/sys/kernel/security/ima/*`, `/sys/kernel/security/evm`, audit-log records, IMA template formats, IMA policy syntax.
- **Keys**: `keyctl(2)` syscall + keyring file format (`/proc/keys`, `/proc/key-users`).

### Capability bits

`include/uapi/linux/capability.h` enumerates capability bits (CAP_CHOWN=0, CAP_DAC_OVERRIDE=1, …, CAP_PERFMON, CAP_BPF, CAP_CHECKPOINT_RESTORE, …). Numeric values + check semantics preserved.

### syscalls

`lsm_get_self_attr(2)`, `lsm_set_self_attr(2)`, `lsm_list_modules(2)` — generic LSM-aware syscalls. Plus per-LSM (`landlock_*`) and the keyctl syscall.

## Requirements

- REQ-1: LSM hook dispatcher invokes hooks identically to upstream — ordering, security-result combination semantics (deny-overrides), and per-hook signature.
- REQ-2: Capability checks (`capable()`, `ns_capable()`, etc.) preserve exact semantics including capability inheritance, file capabilities, and the "ambient" set.
- REQ-3: SELinux preserves type-enforcement, MLS, conditional policies, AVC, security context format, and the binary policy file format (a `policy.31` from upstream loads on Rookery).
- REQ-4: AppArmor preserves profile syntax (path-based DAC + extended attributes), profile load + replace semantics, and policy.bin format.
- REQ-5: SMACK preserves label format and CIPSO/IPv4 + CALIPSO/IPv6 labeling.
- REQ-6: TOMOYO preserves its policy syntax + per-domain profiles + learning mode.
- REQ-7: Yama preserves `ptrace_scope` semantics.
- REQ-8: LoadPin enforces kernel-image-load-origin checks identically.
- REQ-9: Landlock preserves syscall ABI + ruleset semantics; existing sandboxed apps continue to be confined.
- REQ-10: Lockdown preserves "integrity" and "confidentiality" levels.
- REQ-11: Integrity (IMA + EVM) preserves measurement-list format, signature verification, audit records, and TPM PCR extensions.
- REQ-12: Keyring (keyctl) preserves syscall ABI, key types (user, logon, asymmetric, dns_resolver, ceph, cifs, encrypted, trusted, big_key), and `/proc/keys` format.
- REQ-13: safesetid preserves UID/GID restriction policy semantics.
- REQ-14: BPF-LSM preserves the BPF program type + attachment ABI.
- REQ-15: All Tier-3 docs declare their unsafe-block clusters, TLA+ models (LSM hook dispatch is non-trivial), and Kani harnesses.
- REQ-16: SELinux is one of the largest single LSMs (~100K LoC) — the implementation MAY (per Q1) keep it as upstream-C-via-FFI for v0.

## Acceptance Criteria

- [ ] AC-1: `selinux-testsuite` passes with the same set as upstream. (covers REQ-3)
- [ ] AC-2: AppArmor regression tests pass. (covers REQ-4)
- [ ] AC-3: Capability selftests (`tools/testing/selftests/capabilities/`) pass. (covers REQ-2)
- [ ] AC-4: Landlock selftests pass. (covers REQ-9)
- [ ] AC-5: Yama ptrace tests pass. (covers REQ-7)
- [ ] AC-6: IMA + EVM measurement test produces a measurement list verifiable against the TPM. (covers REQ-11)
- [ ] AC-7: Keyring selftests pass. (covers REQ-12)
- [ ] AC-8: Stackable-LSM dispatcher tests pass. (covers REQ-1)
- [ ] AC-9: Lockdown selftests pass. (covers REQ-10)
- [ ] AC-10: BPF-LSM selftests pass. (covers REQ-14)
- [ ] AC-11: `make verify` passes security/ Kani harnesses. (covers REQ-15)
- [ ] AC-12: For each LSM in v0 port set per Q1: a smoke test loads the LSM and verifies its core enforcement. (covers REQ-3 through REQ-14 + REQ-16)

## Architecture

### Layout map

```
.design/security/
  00-overview.md           ← this document
  lsm-framework.md         ← hook dispatcher + stackable LSM + lsm_audit + lsm_notifier + lsm_syscalls + inode + min_addr
  capabilities.md          ← commoncap + cred infrastructure + capability inheritance/file-caps/ambient set
  selinux.md               ← (per Q1: FFI in v0)
  apparmor.md              ← (per Q1: FFI in v0)
  smack.md                 ← (per Q1: FFI in v0)
  tomoyo.md                ← (per Q1: FFI in v0 OR CONFIG=n)
  yama.md                  ← (small; full Rust port in v0)
  loadpin.md               ← (small; full Rust port)
  landlock.md              ← (full Rust port — small surface, growing)
  lockdown.md              ← (small; full Rust port)
  ipe.md                   ← (per Q1: FFI in v0)
  integrity.md             ← IMA + EVM (per Q1: FFI in v0)
  keys.md                  ← keyring + keyctl syscall
  safesetid.md             ← (small)
  bpf-lsm.md               ← cross-ref to kernel/bpf/lsm.md
```

### Cross-references

- `kernel/00-overview.md` — credentials infrastructure (`kernel/cred.c`); audit subsystem (`kernel/audit*`); BPF substrate.
- `fs/00-overview.md` — every fs/ operation has an LSM hook.
- `net/00-overview.md` — every net/ operation has an LSM hook.
- `mm/00-overview.md` — mmap/mprotect have LSM hooks.
- `crypto/00-overview.md` — keys subsystem cross-references asymmetric-keys.
- `00-glossary.md` — `LSM`, `credentials (struct cred)`, `capability`.
- `references/grsec-pax-notes.md` — many GRKERNSEC_* features map to LSM enforcement.

### Rust module organization (informative)

- `kernel::security::lsm` — framework
- `kernel::security::cap` — capability checks
- `kernel::security::keys` — keyring
- per-LSM Rust modules where the LSM is in the v0 Rust port set (yama, loadpin, landlock, lockdown, safesetid)
- FFI'd LSMs use `kernel::security::ffi::<lsm>` thin wrappers around the upstream C

### Locking and concurrency

LSM hook dispatch is hot-path; per-call list traversal is RCU-read-side over the registered hook list. Per-LSM internal locking is per-LSM-specific.

### Error handling

LSMs return `Result<(), KernelError>` for hooks; errors translate to syscall errno (EACCES typically; EPERM for capability-class denies).

## Verification

### Layer 1: Kani SAFETY proofs
- Cred refcount transitions
- LSM hook list RCU traversal
- Per-LSM unsafe blocks (per the Tier-3 docs in the v0 Rust port set)

### Layer 2: TLA+ models
- `models/security/lsm_dispatch.tla` — proves stackable-LSM dispatcher's deny-overrides semantics under concurrent hook calls.
- `models/security/cap_inherit.tla` — proves capability inheritance/file-cap/ambient transitions on exec preserve invariants.

### Layer 3: Kani harnesses
- LSM hook list integrity (no double-registration; ordered by load_priority)
- Capability bitmap operations

### Layer 4: opt-in
- Landlock policy enforcement correctness via Verus
- Capability inheritance arithmetic via Creusot

## Hardening

This subsystem IS the hardening tier. Per `00-overview.md` D6, binding policy lands in `00-security-principles.md` (issue #2). After mm/ + arch/x86 overviews are reviewed, that document is authored grounded in the grsec/PaX catalog plus the existing in-tree LSM enforcement points covered here.

### Critical: grsec absorption is layered — LSMs are NOT replaced

The grsec/PaX reference catalog (`.design/references/grsec-pax-notes.md`) splits cleanly into two layers, and Rookery only absorbs one:

- **Memory-protection + CFI layer** (KERNEXEC, UDEREF, USERCOPY, REFCOUNT, RAP, RANDSTRUCT, AUTOSLAB, PRIVATE_KSTACKS, MEMORY_SANITIZE, …) is **LSM-orthogonal**: it hardens the kernel against memory-corruption exploits at a level below LSM hooks. SELinux / AppArmor / SMACK / Yama / Lockdown / Landlock all see no behavioral difference. **Absorb fully.**
- **Policy-enforcement layer** (GR-RBAC framework, GRKERNSEC_HIDESYM, GRKERNSEC_HIDETASK, GRKERNSEC_TPE, GRKERNSEC_CHROOT_* policy switches, GRKERNSEC_DENYUSB, …) **conflicts with LSM responsibilities**. Historically grsec replaced LSM rather than coexisting; the maintainers were openly hostile to SELinux. Adopting these features by default would break the drop-in compat promise: RHEL/Fedora ship SELinux on; Debian/Ubuntu/SUSE ship AppArmor on; Rookery cannot displace them. **Do NOT absorb as defaults.** Where these have merit as defense-in-depth, expose as sysctl knobs (`00-security-principles.md` will spec the knob set), never as defaults, never as replacements.

This subsystem (`security/`) preserves the LSM framework intact. Every in-tree LSM (SELinux, AppArmor, SMACK, TOMOYO, Yama, LoadPin, Landlock, Lockdown, IPE, IMA+EVM, BPF-LSM) keeps its hooks, its userspace ABI, and its enforcement semantics. Rookery is **grsec-flavored at the memory layer and LSM-respecting at the policy layer.**

### Locked-in row-1 absorption list (decision 2026-05-09)

Per the user-locked decision recorded on issue #1, every PaX/grsec layer-1 feature below is ABSORBED into Rookery. **Default policy: each feature is ON by default and configurable** (revised 2026-05-09). The configurability knob is sysctl, Kconfig, or per-process prctl as appropriate per feature. The four-column table below names: the feature, its Rust expression, its design home (where the implementation is specified), and its default + configurability mechanism.

The row-1 features cleanly split into three operational categories:

- **Type-system-enforced** (the feature is a property of the Rust code itself; "configurability" means escape hatches via attributes / `unsafe` blocks):
- **Sysctl/Kconfig-configurable kernel-internal hardening** (operationally invisible to userspace; default-on with a kernel-side knob to disable for benchmarking or unusual workloads):
- **User-visible per-process restriction** (default-on system-wide would break common userspace JITs; the existing per-process opt-in mechanism — `prctl(PR_SET_MDWE)` — is preserved as the configurable knob):

| PaX/grsec feature | Rust expression | Design home | Default + configurability |
|---|---|---|---|
| **KERNEXEC** (W^X for kernel; non-writable text + non-executable data) | Free in Rust (no JIT, write-protected statics) | `arch/x86/00-overview.md` § paging.md + § kernel-platform.md | **ON, mandatory** (no off switch — invariant in Rust) |
| **UDEREF** (kernel cannot deref user pointers without explicit gate) | Type-encodable: `UserPtr<T>` newtype; `copy_to/from_user` only path | `00-rust-conventions.md` + `lib/00-overview.md` § usercopy.md | **ON, mandatory** (type-system-enforced) |
| **USERCOPY** (whitelist slab caches that may participate in copy_*_user) | Type-encodable: slab-cache type-tagging + compile-time check | `mm/00-overview.md` § slab.md + `lib/00-overview.md` § usercopy.md | **ON, mandatory** (type-system-enforced) |
| **REFCOUNT** (overflow-checked saturating refcounts) | Free via `kernel::sync::Refcount` (upstream existing) | `kernel/00-overview.md` § locking/refcount.md | **ON, mandatory** (type-system-enforced; raw `atomic_t` for refcount discouraged) |
| **RAP/CFI** (forward + backward edge CFI via type signatures) | Per-arch with cross-arch substrate; entry/exit boundary protection | `kernel/00-overview.md` § runtime-codepatching.md + `arch/x86/00-overview.md` § cpu-mitigations.md | **ON by default** (CONFIG_CFI_CLANG=y; Intel CET shadow stack on supporting CPUs); Kconfig-disable for benchmarking or non-CFI compilers |
| **RANDSTRUCT** (randomize layout of internal kernel structs) | Rust `#[repr(Rust)]` default + opt-in `randomize_layout` attribute on internal structs | `00-rust-conventions.md` (struct-layout convention) + per-subsystem opt-in | **ON by default** for marked internal structs; ABI structs (`#[repr(C)]`) NEVER randomized; per-build seed configurable via Kconfig CONFIG_RANDSTRUCT_FULL=y vs `=performance` |
| **AUTOSLAB** (per-callsite slab cache type tagging) | Free via Rust's type system; `core::any::type_name` for forensics | `mm/00-overview.md` § slab.md | **ON, mandatory** (compile-time per-type slab caches) |
| **PRIVATE_KSTACKS** (per-CPU IST stacks; per-task kernel stacks isolated) | Per-CPU IST stacks + per-task kernel stacks | `arch/x86/00-overview.md` § kernel-platform.md + § entry.md | **ON, mandatory** (already required by SMP correctness) |
| **RANDKSTACK** (kernel-stack offset randomization on syscall entry) | Cross-arch infra here; x86 instantiation in entry.md | `kernel/00-overview.md` § syscall-entry-helpers.md + `arch/x86/00-overview.md` § entry.md | **ON by default** (CONFIG_RANDOMIZE_KSTACK_OFFSET=y, runtime sysctl `kernel.randomize_kstack_offset=1` — both flipped from upstream's defaults; userspace-invisible) |
| **MEMORY_SANITIZE** (zero memory on free for slab + page allocators) | Per-cache flag; always-on for `SENSITIVE`-marked types; default-on for all others | `mm/00-overview.md` § slab.md + § page-allocator.md | **ON by default** (boot param `init_on_free=1`, sysctl `vm.zero_on_free=1` — both flipped from upstream's defaults; ~5–10% overhead on alloc-heavy workloads, configurable off per cache or globally) |
| **SIZE_OVERFLOW** (forbid bare arithmetic on size types) | Clippy lint `bare_arithmetic_on_size` (issue #3) + `checked_*`/`saturating_*`/`wrapping_*` discipline | `00-rust-conventions.md` § Forbidden patterns + issue #3 | **ON, mandatory** (compile-time lint deny-level) |
| **CONSTIFY** (auto-const function-pointer-only structs) | Free in Rust — `static FOO: Vtable = …` is `const` by default | `00-rust-conventions.md` | **ON, mandatory** (Rust default) |
| **LATENT_ENTROPY** (gather entropy at boot + runtime, feed RNG) | Boot-time + scheduler-tick entropy harvesting feeding the CSPRNG | `crypto/00-overview.md` § rng.md | **ON by default** (CONFIG_LATENT_ENTROPY=y — flipped from upstream's gcc-plugin default-off; userspace-invisible) |
| **PAGEEXEC** (NX bit enforcement) | Compile-time guarantee (NX is required on x86_64) | `arch/x86/00-overview.md` § paging.md | **ON, mandatory** (architectural; cannot be disabled on x86_64) |
| **NOEXEC** (no-exec for stack/heap/anon-mmap in userspace by default) | mm policy enforced via mmap permissions | `mm/00-overview.md` § mmap.md | **ON by default** for stack + heap + anon-mmap (matches upstream PT_GNU_STACK default); ELF binaries with explicit executable-stack annotation continue to opt back in |
| **NOEXEC strict** (forbid PROT_EXEC even when explicitly requested) | mm policy; would break JITs (V8 / OpenJDK / .NET / BEAM / LuaJIT / PyPy) | `mm/00-overview.md` § mmap.md | **OFF by default at system level**; opt-in per process via `prctl(PR_SET_MDWE)` (existing upstream mechanism); systemd's `MemoryDenyWriteExecute=yes` propagates this. JIT-using processes default to non-strict; security-sensitive processes opt in. |
| **MPROTECT** (per-process W^X — block W→X transitions in mprotect) | mm policy; same JIT-breaking concern as NOEXEC strict | `mm/00-overview.md` § mmap.md | **OFF by default at system level**; opt-in per process via `prctl(PR_SET_MDWE, PR_MDWE_REFUSE_EXEC_GAIN)` (matches existing upstream MDWE). systemd integration via `MemoryDenyWriteExecute=yes`. **Q4 below tracks whether to push for default-on with a JIT-process exemption mechanism.** |
| **DIRECT_CALL / DIRECT_SLS_CALL** (replace indirect calls + Spectre-v2/SLS mitigation) | Retpolines + IBRS + IBPB + eIBRS + LASS where applicable | `arch/x86/00-overview.md` § cpu-mitigations.md | **ON by default** (matches upstream defaults: retpoline + IBRS at boot; LASS on supporting CPUs); kernel-cmdline `mitigations=off` opt-out preserved |
| **RANDMMAP** (strong ASLR for mmap regions) | mm policy | `mm/00-overview.md` § mmap.md | **ON by default** (CONFIG_RANDOMIZE_BASE=y; sysctl `kernel.randomize_va_space=2` — matches upstream); user-controllable via personality(2) per-process for tools that need fixed addresses |
| **ASLR** (strong userspace ASLR for PIE / mmap / stack / exec base) | mm policy | `mm/00-overview.md` § mmap.md + `fs/00-overview.md` § exec-binfmt.md | **ON by default** (sysctl `kernel.randomize_va_space=2`); same per-process opt-out as RANDMMAP |
| **DELAY_FREE_ONE_PAGE** (page quarantine on free to defeat UAF spray) | Page-allocator hardening switch | `mm/00-overview.md` § page-allocator.md | **ON by default** (sysctl `vm.delay_free_pages=1` — flipped from upstream's not-present default; ~few % memory overhead, configurable off per workload) |
| **CLOSE_KERNEL / CLOSE_USERLAND** (SMEP/SMAP-equivalent memory-view switching on user↔kernel transition) | Already handled by SMEP/SMAP (CR4 bits) on x86_64 | `arch/x86/00-overview.md` § entry.md + § paging.md | **ON, mandatory** (architectural; cannot be disabled on x86_64 without CPU support hack) |

### Default-on policy summary

- **Mandatory (cannot disable)**: KERNEXEC, UDEREF, USERCOPY, REFCOUNT, AUTOSLAB, PRIVATE_KSTACKS, SIZE_OVERFLOW, CONSTIFY, PAGEEXEC, CLOSE_KERNEL/CLOSE_USERLAND.
- **Default-on, configurable off**: RAP/CFI, RANDSTRUCT, RANDKSTACK, MEMORY_SANITIZE, LATENT_ENTROPY, NOEXEC (default mode), DIRECT_CALL, RANDMMAP, ASLR, DELAY_FREE_ONE_PAGE.
- **Default-off, configurable on per-process** (the JIT carve-out): NOEXEC strict, MPROTECT-W→X-block. These cannot default-on system-wide without breaking Chrome / Firefox / OpenJDK / .NET / BEAM / LuaJIT / PyPy. Per-process opt-in via `prctl(PR_SET_MDWE)` is preserved; systemd's `MemoryDenyWriteExecute=yes` propagates this for confined services.

Every Tier-3 doc named above gains a **Hardening** section that explicitly cites this table when the doc lands. The deferred `00-security-principles.md` (issue #2) consolidates the cross-cutting policy.

The `00-security-principles.md` doc when authored will:
1. Specify the table above as the binding default-policy reference.
2. Specify the configurability mechanism (sysctl name + boot param + Kconfig + prctl) for each non-mandatory feature.
3. Explicitly **forbid** any row-2 (policy-enforcement) feature from being enabled-by-default; explicitly forbid replacing or short-circuiting the LSM framework.
4. Where a row-2 feature has defense-in-depth value (e.g., kernel-symbol hiding past `kptr_restrict=2`), spec it as an opt-in sysctl that is documented as "complementary to but never substitute for LSM policy."

## Open Questions

<!-- OPEN: Q1 -->
### Q1: Per-LSM Rust port set vs. FFI-to-C
The major LSMs are large: SELinux ~100K LoC, AppArmor ~30K LoC, SMACK ~10K LoC. Re-implementing all in Rust is a multi-year effort.

**Recommendation**:
- **Full Rust port in v0**: yama (small), loadpin (small), landlock (medium, growing — strong fit for Rust), lockdown (small), safesetid (small), keys (medium — keyring core, valuable for Rust), commoncap/capabilities (small but pervasive)
- **FFI in v0**: SELinux (huge, complex), AppArmor (medium-huge), SMACK (medium), TOMOYO (medium), IPE (newer, medium), integrity/IMA + EVM (medium-large)
- **CONFIG=n by default if not in distro defaults**: device_cgroup-as-LSM (legacy v1)

**To resolve**: User confirms split, or names LSMs to move between buckets.
<!-- /OPEN -->

<!-- OPEN: Q2 -->
### Q2: Authorship of 00-security-principles.md
Per `00-overview.md` D6, this Tier-1 doc is authored after Phase A complete + mm/00-overview.md + arch/x86/00-overview.md exist. With this `security/00-overview.md` now drafted, those preconditions are met.

**Recommendation**: Author `00-security-principles.md` (issue #2) immediately after Phase B completes, before Phase C (component-level designs) begins. This locks the binding security policy that all Tier-3 docs cite.

**To resolve**: User confirms timing — eagerly after Phase B, OR deferred until Phase C is partway done.
<!-- /OPEN -->

<!-- OPEN: Q3 -->
### Q3: MPROTECT default-on with JIT-process exemption?
The "default-on, configurable" policy (decision 2026-05-09) flagged MPROTECT-W→X-block as the one feature that cannot default-on system-wide without breaking JIT-using userspace (V8, OpenJDK, .NET, BEAM, LuaJIT, PyPy → Chrome / Firefox / Java apps / .NET apps / Erlang/Elixir apps fail at startup).

The current resolution: default-off system-wide; per-process opt-in via `prctl(PR_SET_MDWE)`. systemd's `MemoryDenyWriteExecute=yes` propagates this for confined services.

**Alternative being considered**: invert the default — MPROTECT-W→X-block default-on system-wide, with a per-process *exemption* mechanism (new prctl `PR_ALLOW_EXEC_GAIN` or analogous) that JIT-using processes call at startup, reflected in their executable's metadata (e.g., a new ELF note `NT_GNU_PROPERTY_X86_FEATURE_NEEDS_EXEC_GAIN`). Distros patch their JIT runtimes to set the exemption; non-JIT processes get the stronger default.

**Recommendation**: ADOPT the inverted default — MPROTECT default-on, with the per-process exemption mechanism. This requires (a) defining a new ELF note + prctl in `00-security-principles.md`, (b) patching JIT runtimes in distro packaging, and (c) accepting that newly-installed JIT software fails until packagers update their build. The transitional period is rough but the steady-state security posture is materially better.

**Alternative-alternative**: keep the current resolution (default-off system-wide; per-process opt-in via existing MDWE). Simpler, no new ABI, but the strong policy is opt-in rather than opt-out — most apps never opt in.

**To resolve**: User picks (a) inverted default with new exemption mechanism (more work, better outcome), (b) keep current MDWE-as-opt-in (simpler), or (c) defer to `00-security-principles.md` authoring time.
<!-- /OPEN -->

## Out of Scope

- Per-LSM detailed policy languages (SELinux .te files, AppArmor profile syntax) — those are userspace concerns; the LSM doc covers the kernel parser.
- Out-of-tree LSMs (SELoad, etc.) — not in our scope.
- 32-bit-only paths.
- Implementation code.
