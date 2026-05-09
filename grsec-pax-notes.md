---
title: "Reference: grsecurity / PaX hardening principles"
tags: ["reference", "security", "backlog", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### why we care

grsecurity + PaX is the most rigorously-engineered Linux hardening patchset. Its features encode hard-won knowledge about real exploit primitives in the kernel attack surface. Many of grsec's invariants are exactly what a Rust-from-scratch kernel can enforce more naturally than C — and recognizing the mapping early prevents us from re-discovering the same hardening categories one CVE at a time.

This file is a reading list, not a binding spec. The eventual binding spec lives in a future foundational doc (provisional name `00-security-principles.md`) and per-subsystem hardening sections.

### patch on disk

- Path: `/home/doll/grsec-6.6.102.patch`
- Size: 14 MB
- Files patched: 4,808
- Target: `linux-6.6.102` (note: older than our v7.1-rc2 baseline; still the most current freely-circulating grsec)
- Acquisition date as observed: 2025-12-21

### feature catalogue (pax_* and grkernsec_*)

Not exhaustive — drawn from a `grep -oE 'PAX_[A-Z_]+'` pass over the patch. Each row maps the upstream feature to its Rust-kernel implication.

### Memory protection (PaX core)

| grsec/PaX feature | What it does | Rust-kernel implication |
|---|---|---|
| `PAX_KERNEXEC` | W^X for kernel: no writable+executable pages, write-protect kernel text/rodata | Largely free — Rust has no JIT, no runtime codegen. Spec it as a hard invariant in the paging design. |
| `PAX_UDEREF` | Kernel cannot dereference userspace pointers without explicit `set_fs`/SMAP-equivalent gate | Encode as Rust types: `UserPtr<T>` is a wrapper; `copy_from_user` is the only way to obtain a `T`; bare deref is impossible without `unsafe`. |
| `PAX_PAGEEXEC` | Real (not segmented) NX bit enforcement | Compile-time guarantee on x86_64 (NX is required). Spec in paging doc. |
| `PAX_MPROTECT` | Per-process W^X for userspace via mprotect restrictions | Userspace policy — enforced by the kernel's `mmap`/`mprotect` paths. Doc in `fs/exec.md` + `mm/mmap.md`. |
| `PAX_NOEXEC` | No-exec for stack, heap, anon-mmap | mm policy. Doc as compat-policy switch. |
| `PAX_PRIVATE_KSTACKS` | Per-CPU kernel stacks isolated from each other (defeats stack-disclosure pivots) | Spec in `arch/x86/kernel-platform.md`. |
| `PAX_RANDKSTACK` | Randomize kernel stack offset per syscall entry | Spec in `arch/x86/entry.md`. |
| `PAX_RANDMMAP` | Strong ASLR for mmap regions in userspace | mm policy. Spec in `mm/mmap.md`. |
| `PAX_ASLR` | Strong userspace ASLR (PIE, mmap, stack, exec base) | mm policy. Spec in `mm/mmap.md` + `fs/exec.md`. |
| `PAX_LATENT_ENTROPY` | Gather entropy from kernel control-flow at runtime, feed RNG | Spec in `crypto/random.md` (CSPRNG seeding). |
| `PAX_MEMORY_SANITIZE` | Zero kernel memory on free (slab + page) | mm/slab policy. Trivial to spec in Rust: enforce `Drop` writes zeros; slab caches register a "sensitive" flag. |
| `PAX_REFCOUNT` | Overflow-checked atomic refcounts; saturating semantics | Free in Rust if we use `core::sync::atomic` correctly + a `Refcount<T>` wrapper that panics on overflow. Spec in `kernel/locking/refcount.md`. |

### Type and memory hygiene

| grsec/PaX feature | What it does | Rust-kernel implication |
|---|---|---|
| `PAX_USERCOPY` | Whitelist slab caches and on-stack ranges that may participate in `copy_to/from_user` | Encode via type tags: `#[user_copyable]` derive on plain-old-data types; slab API checks at compile time. Spec in `mm/slab.md` + `lib/uaccess.md`. |
| `PAX_CONSTIFY_PLUGIN` | gcc plugin that auto-`const`s function-pointer-only structs | Free in Rust — `static FOO: Vtable = Vtable { ... }` is `const`. Spec in conventions doc. |
| `PAX_RANDSTRUCT` | Randomize layout of structs marked with `__randomize_layout` | Rust offers `#[repr(Rust)]` (default) which already gives the compiler layout freedom; we can opt in or out per type. Spec in conventions doc + per-subsystem. |
| `PAX_AUTOSLAB` | Auto-create per-callsite slab caches to limit cross-cache spraying | Spec in `mm/slab.md` as a hardening switch. |
| `PAX_AUTOSLAB_AUTOTYPENAME` | Tag slab caches with type names for forensics + AUTOSLAB | Free in Rust — `core::any::type_name` exists. |
| `PAX_AUTOSLAB_AUTOSTACK` | Like AUTOSLAB but for kstack allocations | Spec in `kernel/fork.md`. |
| `PAX_SIZE_OVERFLOW` | Plugin-detected integer overflow in size computations | Free in Rust — `checked_*` / `saturating_*` / `wrapping_*` arithmetic is explicit. Forbid bare arithmetic on size types in conventions doc. |
| `PAX_INITIFY_PLUGIN` | Move single-use functions/data into `__init` so they get freed after boot | Spec in `init/00-overview.md`. |
| `PAX_INITLOAD` | Anti-tampering for kernel image at boot | Spec in `arch/x86/boot.md`. |
| `PAX_DELAY_FREE_ONE_PAGE` | Quarantine freed pages briefly to defeat UAF spray races | Spec in `mm/page-allocator.md` as a hardening switch. |

### Control-flow integrity

| grsec/PaX feature | What it does | Rust-kernel implication |
|---|---|---|
| `PAX_RAP` (Reuse Attack Protector) | Forward-edge + backward-edge CFI via per-call-site type-signature checks | Goes beyond what mainline kCFI does. Spec in `kernel/cfi.md` (a new doc — does not exist upstream). |
| `PAX_RAP_RET` / `PAX_RAP_XOR` | Backward-edge: encrypt return addresses on the call stack | Like Intel CET shadow stack but cheaper. Spec in `arch/x86/entry.md`. |
| `PAX_DIRECT_CALL` / `PAX_DIRECT_SLS_CALL` | Replace indirect calls with direct calls + Spectre-v2/SLS mitigation | Spec in `arch/x86/cpu-mitigations.md`. |
| `PAX_DLRESOLVE` | Userspace dynamic linker hardening (PLT/GOT) | Userspace concern; spec in `fs/exec.md` ELF loading. |

### Kernel-userspace separation

| grsec/PaX feature | What it does | Rust-kernel implication |
|---|---|---|
| `PAX_CLOSE_KERNEL` / `PAX_CLOSE_USERLAND` | Switch to "kernel mode" / "user mode" memory views (SMEP/SMAP-equivalent for older HW) | Spec in `arch/x86/entry.md` + `arch/x86/paging.md`. |
| `PAX_CLAC` / `PAX_STAC` | x86 SMAP gates (already in mainline) | Spec in `arch/x86/entry.md`. |

### grsec userspace hardening

| grsec feature | What it does | Rust-kernel implication |
|---|---|---|
| `GRKERNSEC_PROC_USER` / `GRKERNSEC_PROC_USERGROUP` | Restrict /proc visibility to authorized users/group | Compat-policy switch. /proc visibility is part of the drop-in compat surface — has to be Kconfig-gated, off by default. Spec in `fs/proc.md`. |
| `GRKERNSEC_HIDESYM` | Hide kernel symbols (kallsyms, /proc/kallsyms, /sys/module/*/sections) from non-root | Same — compat-policy switch. Spec in `kernel/kallsyms.md`. |
| `GRKERNSEC_DMESG` | Restrict dmesg to root | Spec in `kernel/printk.md`. |
| `GRKERNSEC_CHROOT_*` (suite of ~15 toggles) | Block double-chroot, mknod inside chroot, mount, ptrace, sysctl, fchdir, etc. | Spec in `fs/namespace.md` + per-syscall docs. |
| `GRKERNSEC_TPE` | "Trusted Path Execution" — restrict exec to specific paths/uids | Compat-policy switch in `fs/exec.md`. |
| `GRKERNSEC_RBAC` | Role-based access control | Likely out-of-scope for v0; matches functionality of upstream LSM. |
| `GRKERNSEC_HIDETASK` | Hide selected tasks from /proc | Compat-policy switch. |
| `GRKERNSEC_DENYUSB` | Deny new USB devices after a setpoint | Spec in `drivers/usb/00-overview.md`. |

### translation principle for rookery

Three buckets:

1. **Free in Rust** — no design action needed beyond stating the invariant in the conventions doc:
   - PAX_KERNEXEC (W^X for kernel — no JIT to worry about)
   - PAX_CONSTIFY_PLUGIN (Rust statics are const by default)
   - PAX_REFCOUNT (atomic ops never silently overflow if we use the right wrappers)
   - PAX_SIZE_OVERFLOW (forbid bare arithmetic on size_t-equivalents)
   - PAX_AUTOSLAB_AUTOTYPENAME (`core::any::type_name`)

2. **Type-system encodable** — design the abstraction once, get the invariant for free everywhere:
   - PAX_UDEREF (`UserPtr<T>` newtype; `copy_from_user` only path)
   - PAX_USERCOPY (slab cache type tagging + compile-time check)
   - PAX_RANDSTRUCT (Rust's default repr already permits randomization; opt-in attribute)

3. **Explicit per-subsystem policy** — must be specified in the relevant subsystem doc:
   - PAX_RAP / kCFI strictness level
   - PAX_PRIVATE_KSTACKS, PAX_RANDKSTACK
   - PAX_LATENT_ENTROPY
   - PAX_MEMORY_SANITIZE on free
   - All GRKERNSEC_* compat-policy switches
   - Speculative-execution mitigations

### compat tension

A drop-in kernel cannot enable hardening that breaks userspace. Several grsec features (especially the GRKERNSEC_* hiding ones) are compatibility-affecting policy decisions. Rule of thumb: **off-by-default, Kconfig-gated; on-by-default only if the change is invisible to userspace.**

E.g. PAX_KERNEXEC is invisible to userspace → on by default. GRKERNSEC_HIDESYM changes /proc/kallsyms behavior → off by default, gated.

### where this gets formalized

The binding security policy goes in a future foundational doc, provisionally `00-security-principles.md`, written after the foundation Phase A is complete and at least the mm and arch/x86 subsystem overviews exist. Per-subsystem docs cite this reference when their compat surface intersects a grsec feature.

A crosslink issue tracks this work — see issue tagged `security` + `foundation` titled "Author 00-security-principles.md grounded in grsec/PaX catalog".

### related upstream resources (not on disk; for the future doc)

- KSPP (Kernel Self-Protection Project) tracker: kernsec.org/wiki/index.php/Kernel_Self_Protection_Project
- Documentation/process/maintainer-soc.rst-style hardening guides in `Documentation/admin-guide/hw-vuln/`
- `Documentation/security/self-protection.rst` (upstream)
- Linux Hardening (lhf-style) docs and their mapping to upstream Kconfigs

