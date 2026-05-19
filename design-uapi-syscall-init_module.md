---
title: "Tier-5 syscall: init_module(2) — syscall 175"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`init_module(2)` loads a kernel module from a user-provided in-memory ELF image. The kernel copies `len` bytes from `module_image`, parses the ELF, relocates against the running kernel's symbol table, runs the per-module signature check (if `CONFIG_MODULE_SIG_FORCE` or signature requested), allocates module text/data, applies relocations, runs `module_arch_freeing_init`, invokes the module's `init()` function, then publishes the module to `/sys/module/<name>`. The `param_values` C string is parsed and applied as module parameters before `init()` runs.

`init_module(2)` is the legacy module loader; `finit_module(2)` (syscall 313) supersedes it for fd-based loads, allowing kernel-side signature/integrity checks against the source file. `init_module(2)` is retained for in-memory module construction (e.g. kmod, livepatch staging) and for ABI compatibility. Critical for: module-load security, signature enforcement, livepatch, ksplice.

### Acceptance Criteria

- [ ] AC-1: Caller without `CAP_SYS_MODULE`: `-EPERM`.
- [ ] AC-2: `len == 0`: `-EINVAL`.
- [ ] AC-3: `len > MODULE_INIT_LIMIT`: `-EINVAL`.
- [ ] AC-4: `module_image == NULL`: `-EFAULT`.
- [ ] AC-5: Corrupt ELF (bad magic): `-ENOEXEC`.
- [ ] AC-6: Valid signed module, signature required: load succeeds.
- [ ] AC-7: Valid unsigned module, `modules_signed_only=1`: `-ENOKEY`.
- [ ] AC-8: Tampered signature: `-EKEYREJECTED`.
- [ ] AC-9: Module already loaded (same name): `-EEXIST`.
- [ ] AC-10: `__versions` CRC mismatch: `-ELIBBAD`.
- [ ] AC-11: `param_values` exceeds PAGE_SIZE: `-EINVAL`.
- [ ] AC-12: `init()` returns -EIO: load fails with `-EIO`; module not visible in `/proc/modules`.
- [ ] AC-13: `kernel.modules_disabled=1`: subsequent loads `-EPERM`.

### Architecture

```rust
#[syscall(nr = 175, abi = "sysv")]
pub fn sys_init_module(image: UserPtr<u8>, len: usize, params: UserPtr<u8>) -> isize {
    Module::do_init_module(image, len, params)
}
```

`Module::do_init_module(image_ptr, len, params_ptr) -> isize`:
1. Module::check_cap_sys_module(&current_creds())?;     // EPERM
2. if sysctl_modules_disabled { return -EPERM; }
3. if len == 0 || len > MODULE_INIT_LIMIT { return -EINVAL; }
4. let img = Module::copy_image_from_user(image_ptr, len)?;       // EFAULT / ENOMEM
5. Module::verify_signature(&img)?;                                // EKEYREJECTED / ENOKEY
6. let parsed = Module::parse_elf(&img)?;                          // ENOEXEC
7. Module::check_modversions(&parsed)?;                            // ELIBBAD
8. let params = Module::copy_params_from_user(params_ptr)?;        // EFAULT / EINVAL
9. let modules_mutex = MODULES_MUTEX.lock();
10. if Module::find_loaded(&parsed.name).is_some() { return -EEXIST; }
11. let m = Module::layout_and_alloc(&parsed)?;                    // ENOMEM
12. Module::apply_relocations(&parsed, &mut m)?;                   // ENOEXEC
13. Module::apply_params(&mut m, &params)?;                        // EINVAL
14. Module::publish(&m);
15. let rc = Module::run_init(&m);
16. if rc != 0 {
17.   Module::teardown(m);
18.   return rc as isize;
19. }
20. Module::free_init_section(&m);
21. 0

`Module::verify_signature(img) -> Result<()>`:
1. let trailer = b"~Module signature appended~\n";
2. if !img.ends_with(trailer) {
3.   if sysctl_modules_signed_only { return Err(ENOKEY); }
4.   return Ok(()); // unsigned allowed
5. }
6. mod_verify_sig(img)?;                  // EKEYREJECTED on bad sig

### Out of Scope

- ELF relocation per-arch (covered in arch Tier-3 `arch/<arch>/kernel/module.md`).
- Module signature key management (covered in Tier-3 `kernel/module/signing.md`).
- Livepatch staging (covered in Tier-3 `kernel/livepatch.md`).
- Implementation code.

### signature

```c
int init_module(void *module_image, unsigned long len, const char *param_values);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `module_image` | `void *` | in | User pointer to ELF module image bytes. |
| `len` | `unsigned long` | in | Length of the image, in bytes. |
| `param_values` | `const char *` | in | NUL-terminated KEY=VALUE space-separated parameter list, or empty string. |

### return value

| Value | Meaning |
|---|---|
| `0` | Module loaded; `init()` returned 0; module live. |
| `-1` + `errno` | Load failed; module not installed. |

### errors

| errno | Trigger |
|---|---|
| `EPERM` | Missing `CAP_SYS_MODULE` in the current user namespace; module-loading disabled via `/proc/sys/kernel/modules_disabled = 1`. |
| `EFAULT` | `module_image` or `param_values` user pointer faults during copy. |
| `ENOEXEC` | ELF magic / arch / version mismatch; corrupt section table; missing `.modinfo`. |
| `ENOMEM` | Out of memory allocating module text/data/scratch. |
| `EEXIST` | Module with the same name is already loaded (and not requested as livepatch). |
| `EBUSY` | Concurrent load races detected via `modules_mutex`. |
| `EINVAL` | Bad `len` (zero or > `MODULE_INIT_LIMIT`); unparseable `param_values`; bad module-license. |
| `EKEYREJECTED` | Signature verification failed and `CONFIG_MODULE_SIG_FORCE=y` or `modules_signed_only` enforced. |
| `ENOKEY` | Module unsigned and signature required. |
| `ELIBBAD` | Module-version (`__versions`) mismatch and `MODULE_INIT_IGNORE_MODVERSIONS` not in scope. |

### abi surface

```text
__NR_init_module  (x86_64)  = 175
__NR_init_module  (arm64)   = 105
__NR_init_module  (riscv)   = 105
__NR_init_module  (i386)    = 128

/* len ceiling: MODULE_INIT_LIMIT = 64 MiB (configurable). */
/* param_values length capped at PAGE_SIZE - 1. */
/* No flags argument — use finit_module(2) for modern loads. */
```

### compatibility contract

REQ-1: Syscall number is **175** on x86_64. ABI-stable.

REQ-2: Caller MUST have `CAP_SYS_MODULE` in init_user_ns. Capability check happens before any user copy.

REQ-3: `len == 0` or `len > MODULE_INIT_LIMIT` → `-EINVAL` immediately (no copy).

REQ-4: `copy_module_from_user(module_image, len)` reads the entire image into a kvmalloc'd kernel buffer; bounded copy with no partial-read.

REQ-5: Signature verification:
- If `CONFIG_MODULE_SIG_FORCE=y` or sysctl `kernel.modules_signed_only=1`: image MUST carry an appended `~Module signature appended~` trailer with a valid signature against a trusted key. Failure → `-EKEYREJECTED` or `-ENOKEY`.
- Otherwise: signature is best-effort; verified if present, ignored if absent.

REQ-6: ELF parse stages:
- a) Validate ELF header (magic, class, machine, version).
- b) Locate `.modinfo`, `__versions`, `.gnu.linkonce.this_module`.
- c) Compute layout (text, data, rodata, init, exit sizes).
- d) Allocate module memory; relocate; apply symbol resolution.

REQ-7: Module-version check (`__versions`): each imported symbol's CRC must match the running kernel's. Mismatch → `-ELIBBAD` unless module was built with `MODULE_INIT_IGNORE_MODVERSIONS` (only via finit_module flag).

REQ-8: License check: module's `MODULE_LICENSE` MUST be a recognized GPL-compatible string for GPL-only symbol use. Otherwise the module taints the kernel (`TAINT_PROPRIETARY_MODULE`).

REQ-9: Parameter parsing: `param_values` is copy_from_user'd up to `PAGE_SIZE - 1`, then split on whitespace into `KEY=VALUE` pairs and dispatched to each registered `module_param`. Invalid key or out-of-range value → `-EINVAL` (module-load fails entirely, no partial init).

REQ-10: Once memory is laid out and relocations applied, `module->init()` is invoked. Non-zero return from `init()` → module is torn down, returns the init() errno to the caller.

REQ-11: Successful load publishes the module in `/proc/modules`, `/sys/module/<name>/`, and the global module list. From that moment, `delete_module(2)` is the only legitimate removal path.

REQ-12: Concurrent loads of the same module-name are serialized through `modules_mutex`; the loser observes `-EEXIST`.

REQ-13: Per-`/proc/sys/kernel/modules_disabled = 1` (one-shot lockdown): permanently denies all module loads until reboot.

REQ-14: Per-`kernel_lockdown(integrity)`: only signed modules permitted; unsigned → `-EKEYREJECTED`.

REQ-15: Per-`__init` section is freed after `init()` returns success; live module retains only `.text/.data/.rodata/.bss`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_checked_first` | INVARIANT | EPERM returned before any copy_from_user. |
| `len_bounded` | INVARIANT | len ∈ (0, MODULE_INIT_LIMIT]. |
| `signature_enforced_if_required` | INVARIANT | modules_signed_only ⟹ verify_signature must succeed. |
| `no_publish_before_relocate` | INVARIANT | module published only after layout + relocate succeed. |
| `init_failure_teardown` | INVARIANT | init() ≠ 0 ⟹ memory freed, module not in list. |
| `params_atomic_apply` | INVARIANT | any param parse error rolls back entire load. |

### Layer 2: TLA+

`kernel/module-load.tla`:
- States: per-cap-check, per-copy, per-verify, per-parse, per-layout, per-publish, per-init, per-teardown.
- Properties:
  - `safety_unsigned_load_rejected_under_signed_only`.
  - `safety_concurrent_load_serialized`.
  - `safety_init_failure_rolls_back`.
  - `safety_modules_disabled_terminal`.
  - `liveness_load_eventually_completes_or_fails`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_init_module` post: ret 0 ⟹ module in MODULE_LIST | `Module::do_init_module` |
| `verify_signature` post: success ⟹ signature valid against keyring | `Module::verify_signature` |
| `parse_elf` post: parsed.layout ⊆ image bounds | `Module::parse_elf` |
| `run_init` post: ret ≠ 0 ⟹ teardown completes | `Module::run_init` |

### Layer 4: Verus / Creusot functional

Per-`init_module(2)` man-page and per-`kernel/module/main.c` semantic equivalence. LTP `init_module*`, signed-module kselftests pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`init_module(2)` reinforcement:

- **Per-cap-check pre-copy** — defense against per-priv-escalation via image-side smuggling.
- **Per-len ceiling** — defense against per-OOM via giant ELF.
- **Per-signature mandatory under signed-only** — defense against per-unsigned-rootkit load.
- **Per-modules_disabled one-shot** — defense against per-runtime-attack module load post-boot.
- **Per-init-failure rollback** — defense against per-half-installed UAF.
- **Per-modversions strict** — defense against per-ABI-mismatch corruption.

### grsecurity / pax surface

- **PaX UDEREF on `module_image` and `param_values` copy_from_user** — defense against per-image-pointer kernel-deref bug; SMAP forced on the entire `len`-byte copy.
- **CAP_SYS_MODULE strict in init_user_ns only** — even root inside a child userns is denied. GRKERNSEC_CHROOT_CAPS extends this: a chrooted task may never load modules regardless of cap bag.
- **GRKERNSEC_KMOD restrictions** — when set, init_module is permanently disabled after `kernel.modules_disabled=1` is latched at the end of system boot. Live module loads happen only via the early-userspace handover window.
- **GRKERNSEC_MODHARDEN (require signed modules)** — forces `CONFIG_MODULE_SIG_FORCE` semantics regardless of build: every load MUST verify against a built-in trusted keyring. Unsigned image → `-ENOKEY`. Signing keys held by an offline machine; live kernel cannot enroll new keys.
- **kernel_lockdown integrity-mode block** — under `kernel_lockdown=integrity` (or `=confidentiality`), `init_module(2)` is unconditionally denied; only the integrity-checked `finit_module(2)` path is allowed. Returns `-EPERM`.
- **PAX_USERCOPY_HARDEN on ELF chunk copy_from_user** — bounded copy uses whitelisted bounce buffer; no slab leak.
- **PaX KERNEXEC on module text pages** — defense against per-W^X violation; module `.text` mapped RX-only after relocation; `.rodata` mapped RO.
- **GRKERNSEC_HIDESYM on `/proc/modules` and `/sys/module/*/sections/`** — loaded module addresses not exposed to non-root, defending against per-symbol-leak ROP gadget discovery.
- **PAX_REFCOUNT on module refcount** — defense against per-refcount-overflow UAF on rmmod/insmod stress.
- **GRKERNSEC_AUDIT_MODULES** — every successful and every failed init_module logged with PID, comm, module name, signature status, return code. Used to detect attempted rootkit insertion.
- **Module signature key under `kernel_lockdown(integrity)`** — keyring add forbidden; signing key only baked into kernel image.
- **GRKERNSEC_KSTACKOVERFLOW** — defense against per-module-init stack overflow leaking into kernel canary state.
- **GRKERNSEC_DMESG** — module-init `printk` output gated to root + audit.

