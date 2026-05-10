# Foundation: Rookery security principles

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
upstream-paths:
  - Documentation/security/
  - security/Kconfig
  - security/Kconfig.hardening
  - include/uapi/linux/elf.h
  - include/uapi/linux/prctl.h
  - fs/binfmt_elf.c
  - kernel/sys.c
  - mm/mmap.c
  - mm/mprotect.c
  - include/linux/randomize_kstack.h
  - arch/x86/include/asm/nospec-branch.h
  - arch/x86/kernel/cpu/bugs.c
status: draft
-->

## Summary
Binding security policy for Rookery v0. Distills the absorption decisions recorded in `00-overview.md`'s Foundational Decisions table and `security/00-overview.md`'s "Locked-in row-1 absorption list" into a single Tier-1 reference that every Tier-3 component design doc cites in its **Hardening** section. Locks: per-feature default-on policy, configurability mechanisms (sysctl + Kconfig + boot-param + prctl), the new exec-gain ELF note + prctl ABI for JIT-process exemption, GR-RBAC stackable-LSM positioning, and the Hardening-section template Tier-3 docs use.

This document is the **only** place that names exact sysctl/Kconfig/prctl/ELF-note values. Tier-3 docs reference back; they don't redefine. When a Tier-3 doc needs a new knob, it amends this document via a `--kind decision` comment on issue #2.

## Mission

Rookery is a from-scratch Rust rewrite of the Linux kernel targeting full-ABI drop-in compatibility (per `00-overview.md` REQ-1 through REQ-15). Within that compat target, Rookery raises the security baseline to the maximum credible level by:

1. **Memory safety** — using Rust's type system to forbid the largest CVE class in upstream Linux (use-after-free, double-free, OOB write, type confusion). Encoded by `00-rust-conventions.md` and the formal-verification baseline (`00-overview.md` D4).
2. **Defense-in-depth at the memory + control-flow layer** — absorbing PaX/grsec's row-1 features (KERNEXEC, UDEREF, USERCOPY, REFCOUNT, RAP/CFI, RANDSTRUCT, AUTOSLAB, PRIVATE_KSTACKS, RANDKSTACK, MEMORY_SANITIZE, SIZE_OVERFLOW, CONSTIFY, LATENT_ENTROPY, NOEXEC, MPROTECT, DIRECT_CALL, RANDMMAP, ASLR, DELAY_FREE_ONE_PAGE, CLOSE_KERNEL/CLOSE_USERLAND, PAGEEXEC). Default-on with documented configurability per feature.
3. **Stackable LSM enrichment** — preserving the existing in-tree LSM framework intact (SELinux, AppArmor, SMACK, TOMOYO, Yama, LoadPin, Landlock, Lockdown, IPE, IMA+EVM, BPF-LSM) AND adding GR-RBAC (a Rookery-original stackable LSM reimplementing grsec's RBAC framework in Rust, with an empty default policy preserving drop-in compat).
4. **No new attack surface** — Rookery does not add features that aren't security-relevant. Innovation is a v1+ activity.

## Foundational principles (axioms)

These four axioms govern every Tier-3 doc's Hardening section. They are not revisited per-subsystem; they are facts about Rookery.

### Axiom 1: Memory-protection + CFI features absorb fully and default ON

Every PaX/grsec feature in the **row-1 layer** (memory-corruption + control-flow-integrity defenses, LSM-orthogonal) is absorbed and ON by default. The configurability knob exists for benchmarking, debugging, or unusual workloads — not as the expected runtime state.

### Axiom 2: Policy-enforcement features go through GR-RBAC, never default-on system-wide

Every PaX/grsec feature in the **row-2 layer** (policy enforcement: HIDESYM, HIDETASK, TPE, CHROOT-policy-switches, DENYUSB, RBAC) is reimplemented as a policy-loadable setting of the GR-RBAC stackable LSM. The default GR-RBAC policy is empty — no enforcement decisions are made until an operator loads a policy via `gradm`. This preserves drop-in compat: a Rookery boot of an unmodified Debian/Ubuntu/RHEL/Fedora userspace produces identical behavior to upstream Linux.

### Axiom 3: LSM framework is intact; GR-RBAC stacks alongside SELinux/AppArmor

Rookery does NOT replace the LSM framework. Every existing in-tree LSM (SELinux, AppArmor, SMACK, TOMOYO, Yama, LoadPin, Landlock, Lockdown, IPE, IMA+EVM, BPF-LSM) keeps its hooks, its userspace ABI, and its enforcement semantics. GR-RBAC is the 12th LSM, added as a peer that stacks via the standard Linux 5.x+ stackable-LSM machinery. There is **no path** by which a Tier-3 doc can short-circuit, replace, or pre-empt LSM hooks. Where defense-in-depth is needed, GR-RBAC's policy file expresses it; the LSM framework remains the dispatch layer.

### Axiom 4: Userspace-visible behavior changes only through documented opt-out / opt-in mechanisms

Where Rookery's defaults differ from upstream's defaults (RANDKSTACK on, MEMORY_SANITIZE on, LATENT_ENTROPY on, DELAY_FREE_ONE_PAGE on, MPROTECT-W→X-block on with per-process exemption, NOEXEC-strict on with the same exemption), the change is invisible to userspace OR is opt-out via the documented mechanism (sysctl, Kconfig, prctl, ELF note). No silent behavior divergence.

## Requirements

- REQ-1: Every PaX row-1 feature in the absorbed list (`security/00-overview.md` § Locked-in row-1 absorption list) is present in Rookery, defaults ON, and has a documented configurability knob (or is in the "mandatory, no off switch" subset).
- REQ-2: GR-RBAC stackable LSM is built into the default kernel image (CONFIG_LSM_GRBAC=y); `gradm` is shipped as a default userspace utility; the boot-time policy is empty.
- REQ-3: A new userspace ABI — the exec-gain ELF note plus `PR_REQUEST_EXEC_GAIN` prctl — provides the JIT-process exemption from MPROTECT-W→X-block and NOEXEC-strict. Spec'd in this document precisely enough for the implementing instance to build it.
- REQ-4: Sysctl, Kconfig, boot-param, and prctl knob inventory is consolidated in this document; Tier-3 docs reference here, do not redefine.
- REQ-5: Every Tier-3 component design doc gains a **Hardening** section that uses the template defined here, citing this document for any feature it discusses.
- REQ-6: A documented amendment process exists for adding new knobs / changing defaults. Amendments require a `--kind decision` comment on issue #2.
- REQ-7: Every userspace-visible behavior change (where Rookery's default differs from upstream) is enumerated here; Tier-3 docs DO NOT introduce new userspace-visible behavior changes without an amendment to this document.
- REQ-8: No Tier-3 doc may grant itself permission to bypass an LSM hook, short-circuit GR-RBAC enforcement, or introduce policy-enforcement features outside the GR-RBAC LSM. Axioms 2 and 3 are inviolable.
- REQ-9: GR-RBAC's policy engine + gradm are subject to the formal-verification baseline (`00-overview.md` D4); per the design home in `security/grbac/00-overview.md`, the GR-RBAC core gets Layer-1 SAFETY proofs, Layer-2 TLA+ models for policy-evaluation under concurrent fork/exec, and Layer-3 invariant harnesses for policy-graph integrity.
- REQ-10: This document is updated in the same commit that introduces a new knob, changes a default, or resolves an amendment. The frontmatter `baseline-commit` is bumped on each amendment.

## Acceptance Criteria

- [ ] AC-1: A grep for `PaX/grsec row-1 features absorbed` produces a list congruent with `security/00-overview.md` § Locked-in row-1 absorption list (this doc's table is the binding form). (covers REQ-1)
- [ ] AC-2: `make defconfig` on a Rookery checkout produces a `.config` with all the ON-by-default Kconfig values listed in § Knob inventory below. (covers REQ-1, REQ-2, REQ-4)
- [ ] AC-3: A test boots Rookery with default settings, runs Chrome / Firefox / OpenJDK, and confirms each starts successfully — i.e., the JIT-exemption ELF note + prctl mechanism works. (covers REQ-3)
- [ ] AC-4: A test boots Rookery without modifying the JIT runtime's ELF note, runs Chrome, and confirms MPROTECT-W→X-block prevents the JIT (the failure mode is loud and documented; until distros patch their JIT packages, this is the expected behavior). (covers REQ-3, REQ-7)
- [ ] AC-5: A test loads an empty GR-RBAC policy and confirms that an unmodified Debian / Ubuntu / RHEL / Fedora userspace boots and runs identically to upstream Linux. (covers REQ-2, axioms 2-3)
- [ ] AC-6: Every Tier-3 design doc with a Hardening section uses the template from § Per-Tier-3 hardening section template; a CI check enforces the section presence on every doc tagged tier-3. (covers REQ-5)
- [ ] AC-7: A grep over `.design/**/*.md` for "bypass LSM" or "short-circuit LSM" or "skip LSM" returns zero matches. (covers REQ-8)
- [ ] AC-8: GR-RBAC's `make verify` and `make tla` targets pass. (covers REQ-9)
- [ ] AC-9: Every userspace-visible behavior change in this document is enumerated with: the affected interface, the rationale, the opt-out mechanism, and the migration plan. (covers REQ-7)
- [ ] AC-10: The amendment trail in issue #2 (--kind decision comments) matches the version history of this document's `baseline-commit` frontmatter line. (covers REQ-10)

## Architecture

### The two-layer absorption model

The grsec/PaX patchset is split into two layers in `.design/references/grsec-pax-notes.md`. Rookery absorbs both layers but with very different mechanisms.

| Layer | What it does | Conflict with LSMs? | Rookery mechanism |
|---|---|---|---|
| **Row 1: Memory-protection + CFI** | Hardens the kernel against memory-corruption + control-flow-hijack exploits at a layer below LSM hooks | None (orthogonal) | Default-on via Rust types + Kconfig defaults + sysctl defaults; opt-out per feature |
| **Row 2: Policy enforcement** | Adds policy-driven access decisions (TPE, RBAC, HIDESYM-as-default, HIDETASK, etc.) | Yes — competes with SELinux/AppArmor MAC | Reimplemented as the GR-RBAC stackable LSM (CONFIG_LSM_GRBAC=y); features are policy-loadable settings, not default-on system-wide; default policy is empty |

### Locked default-policy table (binding form)

Distilled from `security/00-overview.md` § Locked-in row-1 absorption list. **This is the binding source.** When the two disagree, this document wins; `security/00-overview.md` is amended to match.

#### Mandatory (no off switch — type-system-enforced or architectural)

| Feature | Enforcement mechanism |
|---|---|
| **KERNEXEC** | Compile-time guarantee (Rust `static FOO`s are immutable; no JIT in kernel) + linker script writes `.text + .rodata` as RX/RO |
| **UDEREF** | Type-system enforced via `kernel::user::UserPtr<T>` newtype + `copy_to/from_user` only path |
| **USERCOPY** | Type-system enforced via slab-cache type tagging + compile-time check |
| **REFCOUNT** | Type-system enforced via `kernel::sync::Refcount` (saturating arithmetic) |
| **AUTOSLAB** | Type-system enforced via `kmem_cache_create::<T>()` (per-type slab caches) |
| **PRIVATE_KSTACKS** | Required by SMP correctness; per-CPU IST stacks + per-task kernel stacks |
| **SIZE_OVERFLOW** | Compile-time clippy lint `bare_arithmetic_on_size` (issue #3) |
| **CONSTIFY** | Rust default — `static FOO: Vtable = ...` is `const` |
| **PAGEEXEC** | Architectural — NX bit required on x86_64 |
| **CLOSE_KERNEL / CLOSE_USERLAND** | Architectural — SMEP/SMAP CR4 bits, mandatory on x86_64 |

#### Default-on, configurable off (sysctl / Kconfig / boot-param)

| Feature | Knob (default value) | Off-mechanism |
|---|---|---|
| **RAP/CFI** | `CONFIG_CFI_CLANG=y` (default-on); Intel CET shadow stack on supporting CPUs | Build with non-CFI compiler OR `CONFIG_CFI_CLANG=n`. Per-process: not user-controllable (kCFI is a kernel-side property) |
| **RANDSTRUCT** | `CONFIG_RANDSTRUCT_FULL=y` (default; full randomization of internal structs marked `randomize_layout`); `#[repr(C)]` ABI structs NEVER randomized | `CONFIG_RANDSTRUCT_FULL=n` flips to performance mode (no internal randomization) |
| **RANDKSTACK** | `CONFIG_RANDOMIZE_KSTACK_OFFSET=y`; sysctl `kernel.randomize_kstack_offset = 1` (default-on, **flipped from upstream's default-off**) | Sysctl `kernel.randomize_kstack_offset = 0` |
| **MEMORY_SANITIZE** (zero on free) | Boot param `init_on_free=1` AND sysctl `vm.zero_on_free = 1` (default-on, **flipped from upstream's default-off**); ~5–10% allocator overhead | Boot param `init_on_free=0` OR sysctl `vm.zero_on_free = 0`; per-cache opt-out via the slab-cache `SLAB_NO_ZERO_ON_FREE` flag |
| **LATENT_ENTROPY** | `CONFIG_LATENT_ENTROPY=y` (default-on, **flipped from upstream's gcc-plugin default-off**) | Build with `CONFIG_LATENT_ENTROPY=n` |
| **NOEXEC** (default mode: stack/heap/anon-mmap default non-executable) | Default for ELF binaries without explicit executable-stack note (matches upstream); ELF binaries with PT_GNU_STACK PF_X opt back in | ELF binary's PT_GNU_STACK with PF_X set; `personality(READ_IMPLIES_EXEC)` |
| **DIRECT_CALL / DIRECT_SLS_CALL** (Spectre-v2/SLS mitigation) | Default-on (matches upstream): retpoline + IBRS + IBPB + eIBRS + LASS where applicable | Kernel cmdline `mitigations=off` (matches upstream) |
| **RANDMMAP** | Sysctl `kernel.randomize_va_space = 2` (default-on, matches upstream) | `kernel.randomize_va_space = 0` OR `personality(ADDR_NO_RANDOMIZE)` per process |
| **ASLR** | Sysctl `kernel.randomize_va_space = 2` (matches upstream); ELF PIE binaries get exec-base randomization | Same off mechanism as RANDMMAP |
| **DELAY_FREE_ONE_PAGE** | `CONFIG_DELAY_FREE_PAGES=y` (NEW Kconfig, **not in upstream**); sysctl `vm.delay_free_pages = 1` (default-on); ~few-percent memory overhead | Sysctl `vm.delay_free_pages = 0` |

#### Default-on system-wide with PER-PROCESS exemption (the JIT carve-out)

| Feature | Default behavior | Opt-out mechanism |
|---|---|---|
| **MPROTECT-W→X-block** | Block W→X transitions in `mprotect(2)` for any process whose ELF lacks the exec-gain note | Process's ELF binary carries `NT_ROOKERY_SECURITY_FLAGS` note with the `ROOKERY_SECURITY_NEEDS_EXEC_GAIN` bit set, OR runtime call `prctl(PR_REQUEST_EXEC_GAIN, 1, 0, 0, 0)`; systemd `MemoryDenyWriteExecute=yes` orthogonally tightens for individual services |
| **NOEXEC-strict** (forbid `PROT_EXEC` on stack/heap/anon-mmap even when explicitly requested) | Block `PROT_EXEC` on anonymous + heap mappings | Same exec-gain note + prctl as MPROTECT (one bit covers both) |

The JIT-using userspace stack (Chrome's V8, Firefox's SpiderMonkey, OpenJDK HotSpot, .NET CLR, Erlang BEAM, LuaJIT, PyPy, Python's regex backend if compiled with JIT) MUST carry the note in its packaged ELF binary, OR call the prctl during init before any code generation.

### The exec-gain exemption ABI

#### ELF note (binary-time opt-in)

A new ELF note in the executable's `.note.rookery.security` section grants the exemption at exec time.

**Section name**: `.note.rookery.security`

**Note record format** (per `Documentation/userspace-api/elf.rst` style):

| Field | Size | Value |
|---|---|---|
| `n_namesz` | 4 bytes | 8 (length of "Rookery\0", 8-byte aligned) |
| `n_descsz` | 4 bytes | 4 (length of the flags word) |
| `n_type` | 4 bytes | `NT_ROOKERY_SECURITY_FLAGS` = `0x52455253` ("RERS" — `Rookery RookERy Security`) |
| `n_name` | 8 bytes | `"Rookery\0"` (null-terminated, 4-byte aligned) |
| `n_desc` | 4 bytes | flag bits (see below) |

**Flag bits** (in `n_desc`, little-endian on x86_64):

| Bit | Name | Meaning |
|---|---|---|
| 0 | `ROOKERY_SECURITY_NEEDS_EXEC_GAIN` | Process is allowed to mmap with PROT_EXEC and to mprotect a W mapping to add X |
| 1 | `ROOKERY_SECURITY_NEEDS_RWX_ANON` | Process is allowed to mmap anonymous regions with PROT_WRITE + PROT_EXEC simultaneously (stricter than bit 0; usually JITs only need bit 0) |
| 2-31 | reserved | MUST be zero |

`fs/binfmt_elf.c`'s Rookery counterpart (`fs/00-overview.md` § exec-binfmt.md, Tier 3) reads this note at exec time and sets the per-task `exec_gain_state` field.

#### prctl (runtime opt-in)

For runtime self-tagging — a process that didn't carry the ELF note but wants to JIT before performing untrusted operations:

**`PR_REQUEST_EXEC_GAIN`** = 80 (the next free prctl number after upstream's reserved range; subject to confirmation against upstream prctl assignments at implementation time)

```c
int prctl(PR_REQUEST_EXEC_GAIN, unsigned long flags, 0, 0, 0);
```

`flags` is the same bit-set as the ELF note's `n_desc`. Returns 0 on success, `-EPERM` if the process has called `prctl(PR_SET_NO_NEW_PRIVS, 1, ...)` (preventing privilege gain), `-EBUSY` if the process has already created any executable mapping (you can't opt in retroactively after observing W^X blocked), `-EINVAL` for unknown bits.

**Once granted, exec-gain CANNOT be revoked** within the process. The state is sticky for the task's lifetime; `clone(2)` inherits it; `execve(2)` resets it (re-evaluated against the new ELF's note).

#### Interaction with existing upstream MDWE

`prctl(PR_SET_MDWE, ...)` (existing upstream Memory-Deny-Write-Execute) is preserved unchanged. Its semantics are stricter than Rookery's default:

- Without exec-gain: same effect as upstream's PR_MDWE_REFUSE_EXEC_GAIN
- With exec-gain: `prctl(PR_SET_MDWE, ...)` overrides exec-gain — the process voluntarily tightens itself

systemd's `MemoryDenyWriteExecute=yes` continues to set `PR_SET_MDWE`; it tightens on top of Rookery's default.

### GR-RBAC stackable LSM

Detailed in `security/grbac/00-overview.md` (Tier 3 to be authored — issue #5). This section locks the integration contract:

- **Kconfig**: `CONFIG_LSM_GRBAC=y` in default Kconfig.
- **Stackable-LSM ordering**: GR-RBAC loads LATE in the LSM stack (after capabilities, after the major MAC LSMs SELinux/AppArmor/SMACK). Rationale: GR-RBAC enforcement is additive — it can only deny operations the major LSMs already permitted.
- **Default policy**: empty. No enforcement decisions made until `gradm -L` (learning mode) or `gradm -E` (enforce a loaded policy) is invoked. Drop-in compat preserved per Axiom 2.
- **Userspace tool**: `gradm`, reimplemented in Rust under GPL-2.0 from the open 2018-era grsec patch. Shipped as a default userspace utility (analog to `setenforce` for SELinux, `aa-status` for AppArmor).
- **Policy file format**: derived from grsec-2018 syntax, documented in `security/grbac/policy-file-format.md` (Tier 3). Reuses grsec's RBAC vocabulary (subjects + objects + roles + domains + inheritance + learning) so existing grsec policy files load with mechanical translation.
- **GRKERNSEC_*-as-policy-settings**: HIDESYM, HIDETASK, TPE, CHROOT-policy-switches, DENYUSB are policy-loadable. None default-on — they activate only when a policy file enables them.

### Knob inventory (binding)

This is the consolidated list of every Rookery security knob. Tier-3 docs reference back; they do not introduce new knobs without amending this list.

#### Kconfig

| Symbol | Default | Notes |
|---|---|---|
| `CONFIG_CFI_CLANG` | y | kCFI for indirect calls |
| `CONFIG_RANDSTRUCT_FULL` | y | Full layout randomization for `randomize_layout`-marked internal structs |
| `CONFIG_RANDOMIZE_KSTACK_OFFSET` | y | Per-syscall stack offset randomization |
| `CONFIG_INIT_ON_FREE_DEFAULT_ON` | y | Memory zeroing on free (slab + page) |
| `CONFIG_LATENT_ENTROPY` | y | Boot + runtime entropy harvesting |
| `CONFIG_DELAY_FREE_PAGES` | y | (Rookery-original) Page quarantine on free |
| `CONFIG_LSM_GRBAC` | y | GR-RBAC stackable LSM |
| `CONFIG_FORTIFY_SOURCE` | y | Compile-time bounds checking (matches upstream) |
| `CONFIG_STACKPROTECTOR_STRONG` | y | Stack canaries (matches upstream) |
| `CONFIG_VMAP_STACK` | y | Per-task virtually-mapped kernel stacks (matches upstream) |
| `CONFIG_THREAD_INFO_IN_TASK` | y | thread_info inside task_struct (matches upstream x86_64) |
| `CONFIG_HARDENED_USERCOPY` | y | Runtime usercopy bounds (Rust does this at compile time too) |
| `CONFIG_PAGE_TABLE_CHECK` | y | Page-table walker invariant runtime checks |
| `CONFIG_SHADOW_CALL_STACK` | y | Backward-edge CFI via shadow stack (when CET supported) |
| `CONFIG_DEBUG_LIST` | y | RCU list integrity checks |
| `CONFIG_LSM` | "lockdown,yama,loadpin,safesetid,integrity,selinux,smack,tomoyo,apparmor,bpf,landlock,grbac" | LSM load order (Linux 5.x+ stackable-LSM list); GR-RBAC last |

#### Sysctls

| Sysctl | Default | Notes |
|---|---|---|
| `kernel.randomize_va_space` | 2 | Full ASLR (matches upstream) |
| `kernel.randomize_kstack_offset` | 1 | Per-syscall kstack offset randomization (**flipped from upstream default 0**) |
| `kernel.kptr_restrict` | 2 | Hide all kernel pointers from non-CAP_SYSLOG (matches upstream's stricter mode; default 0 in stock Linux) |
| `kernel.dmesg_restrict` | 1 | dmesg readable only by CAP_SYSLOG (**flipped from upstream default 0**) |
| `kernel.unprivileged_bpf_disabled` | 1 | Unprivileged BPF disabled by default (**flipped**; matches RHEL/Fedora hardening) |
| `kernel.yama.ptrace_scope` | 1 | ptrace restricted to children + CAP_SYS_PTRACE (matches Ubuntu/Fedora default) |
| `vm.zero_on_free` | 1 | Zero memory on free (**flipped from upstream default 0**; ~5–10% allocator overhead) |
| `vm.delay_free_pages` | 1 | Page quarantine on free (Rookery-original sysctl) |
| `vm.mmap_min_addr` | 65536 | Minimum mmap address; defeats null-pointer-deref class (matches upstream) |
| `net.core.bpf_jit_harden` | 2 | Full BPF JIT hardening (constants blinding) |
| `fs.protected_symlinks` | 1 | Restrict symlink-following in sticky-world-writable dirs (matches upstream) |
| `fs.protected_hardlinks` | 1 | Same for hardlinks (matches upstream) |
| `fs.protected_fifos` | 2 | Strict FIFO follow restrictions (**flipped from upstream default 1**) |
| `fs.protected_regular` | 2 | Strict regular-file follow restrictions (**flipped from upstream default 1**) |
| `fs.suid_dumpable` | 0 | No core dumps from SUID binaries (matches upstream) |

#### Kernel command-line parameters

| Parameter | Default | Notes |
|---|---|---|
| `init_on_alloc` | 1 | Zero memory on alloc (**flipped from upstream default 0**) |
| `init_on_free` | 1 | (above; mirrored in sysctl) |
| `mitigations` | (full set) | All CPU mitigations on by default (matches upstream); `mitigations=off` opt-out preserved |
| `slab_nomerge` | y | Disable slab cache merging (defense vs. cross-cache UAF spray) |
| `extra_latent_entropy` | y | Aggressive latent entropy harvesting |
| `module.sig_enforce` | 1 | Enforce kernel module signature verification (matches upstream when CONFIG_MODULE_SIG_FORCE=y) |

#### prctls

| prctl | Number | Notes |
|---|---|---|
| `PR_SET_MDWE` | (existing upstream value, preserved unchanged) | Memory-Deny-Write-Execute; orthogonal to Rookery's default-on MPROTECT-block |
| `PR_REQUEST_EXEC_GAIN` | 80 (Rookery-new; subject to upstream prctl assignment confirmation) | JIT-process exemption from MPROTECT-W→X-block + NOEXEC-strict |
| `PR_SET_NO_NEW_PRIVS` | (existing upstream, preserved) | Prevents PR_REQUEST_EXEC_GAIN — once NoNewPrivs is set, exec-gain returns -EPERM |
| `PR_SET_DUMPABLE` | (existing upstream, preserved) | Coredump permission |

#### ELF notes

| Note | Section | n_type | n_name | Notes |
|---|---|---|---|---|
| `NT_ROOKERY_SECURITY_FLAGS` | `.note.rookery.security` | 0x52455253 | "Rookery\0" | (above) — exec-gain exemption + reserved bits |

### Per-Tier-3 hardening section template

Every Tier-3 component design doc MUST include a **Hardening** section that follows this template. The section explicitly cites this document for every default it relies on.

```markdown
## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

The following row-1 features have part of their implementation in this component:

| Feature | This component's responsibility | Default inherited from |
|---|---|---|
| <feature> | <what this component does> | `00-security-principles.md` § <subsection> |
| ... |

### Row-1 features consumed by this component

The following row-1 features are enforced from elsewhere but constrain this component's design:

- <feature> — <how it constrains; e.g., "all kernel/user copy paths route through `kernel::user::UserPtr<T>`">

### Row-2 / GR-RBAC integration

This component's hooks contribute to GR-RBAC policy decisions via the following LSM hook callbacks:

- <hook name> → <semantic; what GR-RBAC policy this informs>

### Userspace-visible behavior changes

- <none | enumerate per Axiom 4>

### Verification

- Layer 1 SAFETY proofs: <enumerate>
- Layer 2 TLA+ models: <enumerate; especially if the component implements a security primitive>
- Layer 3 invariant harnesses: <enumerate>
- Layer 4 (opt-in): <enumerate if Verus/Creusot/Prusti applies>
```

If a Tier-3 doc has no Hardening section, it cannot graduate from `draft`.

## Verification

This document is a Tier-1 spec; it does not directly own implementation code. The verification artifacts it implies are owned by the relevant Tier-3 docs:

- **GR-RBAC stackable LSM**: `security/grbac/00-overview.md` Layer-1 (SAFETY) + Layer-2 (TLA+ for concurrent fork/exec policy evaluation) + Layer-3 (policy-graph integrity invariants) + Layer-4 (Verus opt-in for the policy-engine evaluator).
- **MPROTECT enforcement**: `mm/00-overview.md` § mmap.md Layer-1.
- **NOEXEC enforcement**: `mm/00-overview.md` § mmap.md Layer-1.
- **ELF note recognition**: `fs/00-overview.md` § exec-binfmt.md Layer-1 + Layer-4 (Creusot opt-in for ELF parser correctness — already declared in fs/00-overview.md).
- **prctl `PR_REQUEST_EXEC_GAIN`**: `kernel/00-overview.md` § task-lifecycle.md Layer-1.
- **Knob inventory consistency**: a CI test reads this document's Knob inventory tables and asserts the live `defconfig` + sysctl defaults match.

## Cross-references

- `00-overview.md` Foundational Decisions § Hardening source row — the keystone where the absorption strategy is summarized.
- `00-rust-conventions.md` — encodes the type-system enforcement of the mandatory features (UDEREF / USERCOPY / REFCOUNT / SIZE_OVERFLOW etc.).
- `references/grsec-pax-notes.md` — the catalog of grsec/PaX features with the two-layer split.
- `security/00-overview.md` — the Tier-2 subsystem doc that hosts the per-feature implementation-home table; this document is the binding distillation.
- `security/grbac/00-overview.md` — Tier-3 hub for the GR-RBAC LSM (issue #5).
- `mm/00-overview.md` § mmap.md — owns mmap/mprotect enforcement; Hardening section per the template above.
- `fs/00-overview.md` § exec-binfmt.md — owns the ELF note reader.
- `kernel/00-overview.md` § task-lifecycle.md — owns the prctl + per-task state.
- `arch/x86/00-overview.md` § cpu-mitigations.md — owns the architecture-side CFI, Spectre-v2, retpoline, IBT, CET implementations.
- `00-glossary.md` — definitions of LSM, capability, credentials, KASLR.
- Issue #2 — the tracking issue for this document. Amendments require `--kind decision` comments here.

## Open Questions

(none — all decisions in this document derive from already-locked decisions on issue #1 or issue #2; new questions go on issue #2 as `--kind plan` and graduate to `--kind decision` when resolved)

## Out of Scope

- Userspace hardening that requires distro coordination beyond JIT-runtime patching (e.g., glibc heap hardening, OpenSSL constant-time discipline) — those are upstream userspace concerns.
- New CPU-mitigation features that don't exist in upstream Linux 7.1.0-rc2 — Rookery does not lead on CPU-vulnerability response in v0; we track upstream.
- Per-syscall hardening beyond what the LSM framework + GR-RBAC already supports (e.g., a separate seccomp-equivalent Rookery framework is not authored; existing seccomp is preserved).
- Replacing the existing module-signing infrastructure (preserved per `kernel/00-overview.md` § module-loading.md).
- Implementation code — `.design/` contains specs only.
