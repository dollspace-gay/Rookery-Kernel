# Tier-3: fs/exec-binfmt — exec, binfmt_elf, binfmt_misc, coredump

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/exec.c
  - fs/binfmt_elf.c
  - fs/binfmt_misc.c
  - fs/binfmt_script.c
  - fs/binfmt_flat.c
  - fs/binfmt_elf_fdpic.c
  - fs/compat_binfmt_elf.c
  - fs/coredump.c
  - fs/kernel_read_file.c
  - include/linux/binfmts.h
  - include/uapi/linux/binfmts.h
  - include/uapi/linux/elf.h
  - include/linux/elfcore.h
-->

## Summary
Tier-3 design for the `execve(2)` and `execveat(2)` userspace-program-loading path. Owns the binfmt dispatcher (a chain of binary-format handlers), `binfmt_elf` (the dominant ELF loader), `binfmt_misc` (generic interpreter registration via `/proc/sys/fs/binfmt_misc/`), `binfmt_script` (`#!` shebang), `binfmt_flat` (no-MMU; deferred), `binfmt_elf_fdpic` (no-MMU PIC), `coredump` (ELF core file generation), and `kernel_read_file` (kernel-internal file-load helper).

**Owns recognition of the new `NT_ROOKERY_SECURITY_FLAGS` ELF note** — the binary-time opt-in for MPROTECT-W→X-block + NOEXEC-strict exemption per `00-security-principles.md`. This Tier-3 reads the note at exec time and sets `task->exec_gain_state`.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| `execve` core | `fs/exec.c` |
| ELF loader | `fs/binfmt_elf.c` |
| Compat (32-on-64) ELF loader | `fs/compat_binfmt_elf.c` |
| Misc interpreter registration | `fs/binfmt_misc.c` |
| Shebang loader | `fs/binfmt_script.c` |
| No-MMU formats (deferred) | `fs/binfmt_flat.c`, `fs/binfmt_elf_fdpic.c` |
| Core-dump generation | `fs/coredump.c` |
| Kernel file-load helper (firmware, etc.) | `fs/kernel_read_file.c` |
| ABI headers | `include/linux/binfmts.h`, `include/uapi/linux/binfmts.h`, `include/uapi/linux/elf.h`, `include/linux/elfcore.h` |

## Compatibility contract

### Syscalls

`execve`, `execveat`. Plus userspace-visible `/proc` interfaces: `binfmt_misc` registration via `/proc/sys/fs/binfmt_misc/register`.

### ELF format (`include/uapi/linux/elf.h`)

ELF64 little-endian (x86_64): `EM_X86_64=62`, `EI_CLASS=ELFCLASS64`, `EI_DATA=ELFDATA2LSB`. ELF32 (`EM_386=3`) consumed by compat layer for 32-bit-on-64-bit.

Supported ELF types: ET_EXEC (static), ET_DYN (PIE / shared lib + main), ET_CORE (consumed by `coredump.c` output).

### `auxv` (auxiliary vector)

The aux vector passed to userspace at exec. Tags identical to upstream:
`AT_NULL=0`, `AT_IGNORE=1`, `AT_EXECFD=2`, `AT_PHDR=3`, `AT_PHENT=4`, `AT_PHNUM=5`, `AT_PAGESZ=6`, `AT_BASE=7`, `AT_FLAGS=8`, `AT_ENTRY=9`, `AT_NOTELF=10`, `AT_UID=11`, `AT_EUID=12`, `AT_GID=13`, `AT_EGID=14`, `AT_PLATFORM=15`, `AT_HWCAP=16`, `AT_CLKTCK=17`, `AT_FPUCW=18`, `AT_DCACHEBSIZE=19`, `AT_ICACHEBSIZE=20`, `AT_UCACHEBSIZE=21`, `AT_BSE=22`, `AT_BASE_PLATFORM=24`, `AT_RANDOM=25`, `AT_HWCAP2=26`, `AT_RSEQ_FEATURE_SIZE=27`, `AT_RSEQ_ALIGN=28`, `AT_HWCAP3=29`, `AT_HWCAP4=30`, `AT_EXECFN=31`, `AT_SYSINFO=32`, `AT_SYSINFO_EHDR=33`, `AT_L1I_*`, `AT_L1D_*`, `AT_L2_*`, `AT_L3_*` (cache info), `AT_MINSIGSTKSZ=51`.

Identical numeric values + content semantics.

### binfmt_misc registration ABI

`/proc/sys/fs/binfmt_misc/register` accepts a string of the form `:name:type:offset:magic:mask:interpreter:flags` per `Documentation/admin-guide/binfmt-misc.rst`. Identical semantics so qemu-user-static + Java-class-loader registrations work unchanged.

### Coredump format

`fs/coredump.c` produces ET_CORE ELF files consumed by `gdb`, `lldb`, `crash`, `drgn`. Format: ELF header + program headers + notes (NT_PRSTATUS, NT_PRPSINFO, NT_PRFPREG, NT_AUXV, NT_FILE, NT_SIGINFO, NT_X86_XSTATE, NT_X86_SHSTK, ...) + segment data. Byte-identical to upstream.

`/proc/sys/kernel/core_pattern` controls dump path/pipe; identical.

### Exec-gain ELF note (Rookery-original)

Per `00-security-principles.md` § exec-gain ABI:

**Section**: `.note.rookery.security`

**Note record**:
- `n_namesz` = 8 (length of "Rookery\0", 8-byte aligned)
- `n_descsz` = 4 (flag word)
- `n_type` = 0x52455253 (`NT_ROOKERY_SECURITY_FLAGS`)
- `n_name` = "Rookery\0"
- `n_desc` = flag bits (bit 0 = ROOKERY_SECURITY_NEEDS_EXEC_GAIN, bit 1 = ROOKERY_SECURITY_NEEDS_RWX_ANON, bits 2-31 reserved)

`fs/binfmt_elf.c` analog reads this note during exec. If found and bits set, sets `current->exec_gain_state` accordingly. Cross-ref `kernel/task-lifecycle.md` for the per-task state field.

If bits 2-31 of the flag word are non-zero (reserved), exec is rejected with `-EINVAL` (forward-compat: future expansion of the flag word is enforced).

### Coredump exec-gain interaction

If a process invokes `prctl(PR_REQUEST_EXEC_GAIN)` and later crashes, the resulting core file's NT_AUXV (or a new note `NT_ROOKERY_EXEC_GAIN_STATE`) records the exec_gain_state at crash time. Forensic tools can audit JIT processes' security posture.

## Requirements

- REQ-1: `execve` + `execveat` syscalls have byte-identical entry/exit ABI per upstream.
- REQ-2: ELF loader (`binfmt_elf`) supports all upstream-supported ELF types (ET_EXEC, ET_DYN); HWCAP probing for AT_HWCAP/HWCAP2/HWCAP3 byte-identical.
- REQ-3: `auxv` content + ordering identical; `AT_RANDOM`, `AT_PLATFORM`, `AT_BASE_PLATFORM`, `AT_SYSINFO_EHDR` (vDSO base) populated identically.
- REQ-4: `binfmt_misc` registration via `/proc/sys/fs/binfmt_misc/register` has byte-identical syntax + semantics; existing distro registrations (qemu-user-static, Java, mono, dotnet) work unchanged.
- REQ-5: `binfmt_script` (`#!` shebang) syntax + interpreter resolution byte-identical (incl. multi-arg interpreter line per upstream's `__SCRIPT_INTERPRETER` parsing).
- REQ-6: Coredump format byte-identical: same notes (incl. NT_X86_XSTATE for FPU state, NT_X86_SHSTK for shadow stack), same segment ordering, same naming convention.
- REQ-7: `core_pattern` sysctl: pipe-to-program, file-template, host-id, comm, etc. all parseable per upstream.
- REQ-8: `setuid`/`setgid` exec semantics (capability transitions, securebits, RES_TIME accumulation) byte-identical.
- REQ-9: PT_GNU_STACK ELF program header recognition: PF_X bit gates whether stack is executable. PT_GNU_RELRO + PT_GNU_PROPERTY recognized identically.
- REQ-10: PT_GNU_PROPERTY parsing: NT_GNU_PROPERTY_X86_FEATURE_1_SHSTK / IBT bits trigger `arch_prctl(ARCH_SHSTK_ENABLE)` for the new task. Identical to upstream.
- REQ-11: **`NT_ROOKERY_SECURITY_FLAGS` note recognition**: read at exec time; `current->exec_gain_state` set per the flag bits; reject exec if reserved bits non-zero.
- REQ-12: NoNewPrivs (`PR_SET_NO_NEW_PRIVS`) exec semantics preserved: cannot gain privileges via exec; suid bits ignored; cap bounding set restricts.
- REQ-13: ELF file size limits, segment count limits, etc. per upstream Kconfig (CONFIG_MAX_USER_RT_PRIO, ELF_HASH_BUCKET_LIMIT, …).
- REQ-14: kernel_read_file: kernel-internal helper for reading firmware, signed module images, and ima-policy files. Returns `Result<Vec<u8>, Error>`. ABI for `__init_module(2)` cross-ref `kernel/00-overview.md` § module-loading.md.
- REQ-15: Hardening section per `00-security-principles.md` template.
- REQ-16: Layer-4 functional correctness via Creusot for the ELF parser (binfmt_elf parser correctness — declared in `fs/00-overview.md` Layer-4 candidates).

## Acceptance Criteria

- [ ] AC-1: `strace -e trace=execve` of a typical shell startup matches upstream byte-for-byte. (covers REQ-1)
- [ ] AC-2: Every `/usr/bin/*` ELF binary on a stock Ubuntu/Fedora distro executes correctly under Rookery. (covers REQ-2)
- [ ] AC-3: A test program reads `/proc/self/auxv`; field-by-field comparison against upstream is identical (modulo addresses subject to KASLR). (covers REQ-3)
- [ ] AC-4: A `qemu-user-static`-registered binary (e.g., a foreign-arch binary running on x86_64 host) executes correctly via the binfmt_misc dispatch. (covers REQ-4)
- [ ] AC-5: A `#!/bin/sh` script + a multi-arg interpreter line script both execute correctly. (covers REQ-5)
- [ ] AC-6: A SIGSEGV core dump's notes (decoded by `eu-readelf -n`) match upstream's notes; `gdb` opens both core files identically. (covers REQ-6)
- [ ] AC-7: `core_pattern=|/usr/lib/systemd/systemd-coredump %P %u %g %s %t %e` works correctly (test crashes a process; systemd-coredump receives the dump via pipe). (covers REQ-7)
- [ ] AC-8: A setuid binary's effective UID change at exec is identical to upstream; capability transitions follow `cap_bset & cap_bounding & ~cap_ambient` algebra. (covers REQ-8)
- [ ] AC-9: An ELF binary with PT_GNU_STACK PF_X has executable stack; one without has non-executable stack. (covers REQ-9)
- [ ] AC-10: A binary with `-fcf-protection=full` (sets NT_GNU_PROPERTY_X86_FEATURE_1_SHSTK) has CET shadow stack enabled at exec. Verifiable via `prctl(ARCH_SHSTK_STATUS)` post-exec. (covers REQ-10)
- [ ] AC-11: A test binary built with the `NT_ROOKERY_SECURITY_FLAGS` note (bit 0 set) exec'd: subsequent `mprotect(W→X)` succeeds. Same binary stripped of the note: mprotect returns EACCES. After `execve` to a no-note binary: exec_gain_state resets to Disabled. (covers REQ-11)
- [ ] AC-12: A NoNewPrivs-tagged process exec'ing a setuid binary does NOT gain euid; matches upstream behavior. (covers REQ-12)
- [ ] AC-13: An ELF with > MAX_ARG_PAGES of cmdline + env returns E2BIG identically. (covers REQ-13)
- [ ] AC-14: `request_firmware` (kernel-side, calls `kernel_read_file`) loads a curated firmware blob successfully on Rookery. (covers REQ-14)
- [ ] AC-15: Hardening section present and follows template. (covers REQ-15)
- [ ] AC-16: Creusot proof of `binfmt_elf` parser correctness compiles and verifies. (covers REQ-16)

## Architecture

### Rust module organization

- `kernel::fs::exec` — `execve` core
- `kernel::fs::binfmt::Binfmt` — binfmt-handler trait + chain dispatch
- `kernel::fs::binfmt::elf` — ELF loader
- `kernel::fs::binfmt::elf::compat32` — compat ELF32 loader
- `kernel::fs::binfmt::misc` — `/proc/sys/fs/binfmt_misc/` interpreter registry
- `kernel::fs::binfmt::script` — shebang dispatcher
- `kernel::fs::binfmt::elf::note::rookery` — `NT_ROOKERY_SECURITY_FLAGS` parsing + exec_gain_state setup
- `kernel::fs::coredump` — core-file generation
- `kernel::fs::kernel_read_file` — kernel-side helper
- `kernel::fs::auxv::Auxv` — auxv builder

### Key data structures

- `linux_binprm` — exec-time scratch struct holding partially-loaded program state (path, mm, file, argc, argv, envp, secureexec, ...)
- `binfmt_handler` — vtable: `load_binary`, `load_shlib`, `core_dump`
- `auxv_entry` — pair of (AT_*, value) appended to user stack at exec

### Locking and concurrency

- `cred_guard_mutex`: held during `execve` to prevent racing setresuid + ptrace attach. Already in upstream `signal_struct`; preserved.
- `mm->mmap_lock`: write-side held during exec setup (clearing old mm, mapping new).
- File reads (`vfs_read_iter` on the binary) follow file-table refcount + RCU.

### Error handling

- `Err(EACCES)` — file lacks exec permission; LSM denied
- `Err(ENOENT)` — file not found; interpreter not found
- `Err(ENOEXEC)` — file isn't a recognized binary format
- `Err(ETXTBSY)` — file is currently being written
- `Err(EINVAL)` — bad ELF header / **NT_ROOKERY_SECURITY_FLAGS reserved bits set**
- `Err(EAGAIN)` — RLIMIT_NPROC reached
- `Err(E2BIG)` — argc + envc too large
- `Err(EFAULT)` — bad userspace pointer in argv/envp

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| ELF header parsing (size + offset bounds) | `kani::proofs::fs::binfmt_elf::header_parse_safety` |
| Program-header table iteration | `kani::proofs::fs::binfmt_elf::phdr_iter_safety` |
| Section-header iteration | `kani::proofs::fs::binfmt_elf::shdr_iter_safety` |
| `.note.*` section parsing (incl. NT_ROOKERY_SECURITY_FLAGS) | `kani::proofs::fs::binfmt_elf::note_parse_safety` |
| binfmt_misc registration string parsing | `kani::proofs::fs::binfmt_misc::parse_safety` |
| Coredump segment write | `kani::proofs::fs::coredump::write_safety` |
| auxv build (writes to user stack) | `kani::proofs::fs::auxv::build_safety` |

### Layer 2: TLA+ models

- `models/fs/exec_cred_swap.tla` (NEW) — proves `execve`'s cred swap is atomic w.r.t. concurrent ptrace attach + setresuid; cred_guard_mutex ordering.
- `models/arch/x86/exec_gain_propagation.tla` (inherited) — co-owned with `kernel/task-lifecycle.md` and `mm/mmap.md`.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| binfmt handler chain | Iteration order matches registration order; first-handler-that-claims wins | `kani::proofs::fs::binfmt::chain_invariants` |
| binfmt_misc interpreter table | No duplicate registration with same magic; registration string parse round-trips | `kani::proofs::fs::binfmt_misc::table_invariants` |

### Layer 4: Functional correctness (declared in `fs/00-overview.md` REQ-9 Layer-4 list)

- **ELF loader correctness** via Creusot — proves: for any well-formed ELF file (per the format spec), the loader produces a `mm_struct` with VMAs covering the LOAD segments at the correct addresses with correct permissions. Tractable; high-value (the ELF parser is a frequent attack surface).
- **NT_ROOKERY_SECURITY_FLAGS parser correctness** via Verus — proves: given a byte buffer purporting to be a note section, the parser either rejects it cleanly or returns a flag value with reserved bits zeroed. (Defends against forgery + future-incompat bypass.)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MPROTECT-W→X-block** (binary-time opt-in) | Reads `NT_ROOKERY_SECURITY_FLAGS` ELF note at exec; sets `current->exec_gain_state` per bit 0 (NEEDS_EXEC_GAIN); rejects exec if reserved bits non-zero | § Default-on system-wide with per-process exemption |
| **NOEXEC-strict** (binary-time opt-in) | Same note's bit 1 (NEEDS_RWX_ANON) sets the additional bit | § Default-on system-wide with per-process exemption |
| **NOEXEC** (default mode) | PT_GNU_STACK PF_X bit honored; default non-executable stack | § Default-on configurable off |
| **ASLR** (PIE exec-base randomization) | ET_DYN binaries get randomized exec-base via `arch_mmap_rnd` | § Default-on configurable off |
| **CET shadow stack** (PT_GNU_PROPERTY trigger) | NT_GNU_PROPERTY_X86_FEATURE_1_SHSTK bit triggers shadow-stack enable for the new task | (cross-ref `arch/x86/kernel-platform.md`) |

### Row-1 features consumed by this component

- **UDEREF**: argv/envp pointers from userspace handled via `UserPtr<...>`
- **SIZE_OVERFLOW**: ELF segment-size + auxv-size accumulation uses checked operators
- **AUTOSLAB**: `linux_binprm` allocated via `KmemCache::<LinuxBinPrm>::new()`
- **MEMORY_SANITIZE**: binprm cache freed objects zeroed
- **KERNEXEC**: this component's text is RX/RO

### Row-2 / GR-RBAC integration

`execve` is a major LSM-hook nexus:
- `security_bprm_creds_for_exec(bprm)` — pre-load credential setup
- `security_bprm_check(bprm)` — per-handler permission check
- `security_bprm_creds_from_file(bprm, file)` — load creds from file (LSM file label)
- `security_bprm_committing_creds(bprm)` — about-to-commit
- `security_bprm_committed_creds(bprm)` — post-commit

GR-RBAC (per its loaded policy) implements TPE (Trusted Path Execution) via these hooks. Default-empty policy preserves drop-in compat.

### Userspace-visible behavior changes

Per Axiom 4 of `00-security-principles.md`:
- **`NT_ROOKERY_SECURITY_FLAGS` ELF note is the binary-time form of the JIT exemption**. Distros patch JIT runtimes' build to set this note. Documented in `00-security-principles.md` § exec-gain ABI.
- **Reserved bits non-zero in the note → exec rejected with EINVAL**. Forward-compatible: future expansion of the flag word is enforced; binaries built against an older Rookery don't bypass new restrictions.
- All other behaviors match upstream.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on `copy_strings_kernel` / `copy_string_kernel` and on every `get_user_pages_remote` page-copy into the new mm.
- **PAX_KERNEXEC** — W^X for any executable mapping; `setup_arg_pages` cannot stack-grow into a region with `VM_EXEC|VM_WRITE`.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization across `execve` / `execveat` so initial-stack canary placement is unpredictable.
- **PAX_REFCOUNT** — saturating refcount on `struct linux_binfmt`, on `struct linux_binprm`, and on the per-binfmt module reference so a fast-path `execve` flood cannot wrap.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `linux_binprm.argv`/`envp` page array and for the scratch bprm pages on failure.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access; `get_user_arg_ptr` always uses the user-access helpers.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `linux_binfmt.load_binary` / `load_shlib` / `core_dump` vtables; binfmt registration validates the table signature before linking into `formats`.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding in `/proc/<pid>/auxv`, `/proc/<pid>/stat` start_code/end_code.
- **GRKERNSEC_DMESG** — syslog restriction on bprm/coredump diagnostics that otherwise expose mapping addresses.
- **`AT_SECURE` auxv** — `bprm->secureexec` propagates into `AT_SECURE`; glibc honors this to suppress `$ORIGIN`, `LD_*`, and other untrusted env on suid/sgid/caps-elevated exec.
- **`PR_SET_NO_NEW_PRIVS` (NNP) gate** — `bprm_check_security` refuses any privilege transition (suid, file caps, LSM) when `current->no_new_privs` is set; sandboxes use this to make `execve` strictly non-elevating.
- **suid-bit handling** — `bprm_fill_uid` zeroes `euid`/`egid` raise when the executable is on `nosuid`, when ptraced under `MAY_PTRACE`, or when the calling user-ns is non-init; mismatch is silent demotion not silent escalation.
- **`NT_ROOKERY_SECURITY_FLAGS` reserved-bits enforcement** — non-zero reserved bits in the JIT-exemption note force `-EINVAL` so a future flag word cannot be silently bypassed by a stale binary.
- **`binfmt_misc` register requires `CAP_SYS_ADMIN`** — interpreter-table mutation is restricted to the init user-namespace so user-namespace-confined containers cannot register attacker-controlled MIME handlers.
- **Coredump pipe `CAP_SYS_ADMIN`** — `core_pattern = "|..."` requires capable in init userns; non-init namespaces fall back to file-only dumps.

Per-doc rationale: `execve` is the privilege-transition syscall; every other defense in the kernel depends on the binfmt path correctly clearing `secureexec`, propagating `AT_SECURE`, honoring NNP, and rejecting malformed ELF before the new mm is committed. PaX/grsec reinforcement here is what makes `nosuid` mounts, capability bounding sets, and seccomp confinement actually load-bearing rather than advisory.

## Open Questions

(none — exec/binfmt syscalls are exhaustively specified; the new ELF note ABI is locked in `00-security-principles.md`)

## Out of Scope

- No-MMU formats (`binfmt_flat`, `binfmt_elf_fdpic`) — out per `arch/x86/00-overview.md` D1 (no 32-bit-only kernel)
- 32-bit kernel exec paths
- Implementation code
