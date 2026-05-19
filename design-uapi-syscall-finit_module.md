---
title: "Tier-5 syscall: finit_module(2) — syscall 313"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`finit_module(2)` is the modern fd-based module loader: instead of an in-memory image (as in `init_module(2)`) it takes an open file descriptor pointing at the module file on disk. The kernel can therefore measure the file inode for IMA/IPE integrity, fetch the signature from the file directly, and pass the file through a decompression path (`MODULE_INIT_COMPRESSED_FILE`). Three flag bits coarsely relax safety:
- `MODULE_INIT_IGNORE_MODVERSIONS` — skip per-symbol CRC check.
- `MODULE_INIT_IGNORE_VERMAGIC` — skip kernel vermagic string check.
- `MODULE_INIT_COMPRESSED_FILE` — the fd points at a compressed module; kernel must decompress.

Modprobe / kmod / systemd-modules-load use `finit_module(2)` exclusively on modern systems. Critical for: signed-module enforcement, IMA integrity, livepatch, secure-boot chain.

### Acceptance Criteria

- [ ] AC-1: `finit_module(fd, "", 0)` with valid signed module: returns 0.
- [ ] AC-2: `fd == -1`: `-EBADF`.
- [ ] AC-3: `fd` from `open("/etc/hostname", O_RDONLY)` (not ELF): `-ENOEXEC`.
- [ ] AC-4: `flags = 0x8` (unknown bit): `-EINVAL`.
- [ ] AC-5: `flags = COMPRESSED_FILE`, zstd-compressed module: decompresses and loads.
- [ ] AC-6: `flags = COMPRESSED_FILE`, corrupt compressed stream: `-EBADMSG`.
- [ ] AC-7: `flags = IGNORE_MODVERSIONS` under hardened mode: `-EINVAL`.
- [ ] AC-8: `flags = IGNORE_VERMAGIC` under hardened mode: `-EINVAL`.
- [ ] AC-9: Caller without `CAP_SYS_MODULE`: `-EPERM`.
- [ ] AC-10: Module already loaded (same name): `-EEXIST`.
- [ ] AC-11: Unsigned module, `modules_signed_only=1`: `-ENOKEY`.
- [ ] AC-12: Tampered signature: `-EKEYREJECTED`.
- [ ] AC-13: IMA appraisal fails: `-EACCES`.
- [ ] AC-14: Two concurrent finit_module same module: one succeeds, other returns same result.

### Architecture

```rust
#[syscall(nr = 313, abi = "sysv")]
pub fn sys_finit_module(fd: i32, params: UserPtr<u8>, flags: i32) -> isize {
    Module::do_finit_module(fd, params, flags as u32)
}
```

`Module::do_finit_module(fd, params_ptr, flags) -> isize`:
1. Module::check_cap_sys_module(&current_creds())?;          // EPERM
2. if sysctl_modules_disabled { return -EPERM; }
3. const VALID = MODULE_INIT_IGNORE_MODVERSIONS | MODULE_INIT_IGNORE_VERMAGIC | MODULE_INIT_COMPRESSED_FILE;
4. if (flags & !VALID) != 0 { return -EINVAL; }
5. if kernel_lockdown_active(INTEGRITY) && (flags & (IGNORE_MODVERSIONS | IGNORE_VERMAGIC)) != 0 { return -EINVAL; }
6. let file = fdget(fd)?;                                    // EBADF
7. if !file.mode.contains(FMODE_READ) { return -EBADF; }
8. security_kernel_read_file(&file, READING_MODULE)?;        // EACCES / EPERM via IMA
9. let raw = Module::read_file_into_kvmalloc(&file)?;
10. let image = if (flags & MODULE_INIT_COMPRESSED_FILE) != 0 {
11.   Module::decompress(&raw)?                              // EBADMSG / ENOMEM
12. } else {
13.   raw
14. };
15. if image.len() > MODULE_INIT_LIMIT { return -EINVAL; }
16. Module::verify_signature(&image)?;                       // EKEYREJECTED / ENOKEY
17. let parsed = Module::parse_elf(&image)?;                 // ENOEXEC
18. if (flags & MODULE_INIT_IGNORE_VERMAGIC) == 0 {
19.   Module::check_vermagic(&parsed)?;                      // ENOEXEC
20. }
21. if (flags & MODULE_INIT_IGNORE_MODVERSIONS) == 0 {
22.   Module::check_modversions(&parsed)?;                   // ELIBBAD
23. }
24. let params = Module::copy_params_from_user(params_ptr)?; // EFAULT / EINVAL
25. let mutex = MODULES_MUTEX.lock();
26. Module::idempotent_dispatch(&parsed.name, || {
27.   if Module::find_loaded(&parsed.name).is_some() { return -EEXIST; }
28.   let m = Module::layout_and_alloc(&parsed)?;
29.   Module::apply_relocations(&parsed, &mut m)?;
30.   Module::apply_params(&mut m, &params)?;
31.   Module::publish(&m);
32.   let rc = Module::run_init(&m);
33.   if rc != 0 { Module::teardown(m); return rc as isize; }
34.   Module::free_init_section(&m);
35.   0
36. })

`Module::decompress(raw) -> Result<Vec<u8>>`:
1. match magic(raw) {
2.   ZSTD_MAGIC => zstd_decompress(raw),
3.   XZ_MAGIC   => xz_decompress(raw),
4.   GZ_MAGIC   => gz_decompress(raw),
5.   _          => Err(EBADMSG),
6. }

### Out of Scope

- Compressor implementations (covered in Tier-3 `lib/decompress_*.md`).
- IMA policy engine (covered in Tier-3 `security/integrity/ima.md`).
- ELF relocation per-arch (covered in arch Tier-3 docs).
- Implementation code.

### signature

```c
int finit_module(int fd, const char *param_values, int flags);
```

```c
#define MODULE_INIT_IGNORE_MODVERSIONS  0x0001
#define MODULE_INIT_IGNORE_VERMAGIC     0x0002
#define MODULE_INIT_COMPRESSED_FILE     0x0004
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open fd referring to the module file (must be readable; O_PATH not sufficient). |
| `param_values` | `const char *` | in | NUL-terminated KEY=VALUE space-separated parameter string. |
| `flags` | `int` | in | Bitwise OR of `MODULE_INIT_*`; unknown bits → `-EINVAL`. |

### return value

| Value | Meaning |
|---|---|
| `0` | Module loaded; `init()` returned 0. |
| `-1` + `errno` | Load failed; module not installed. |

### errors

| errno | Trigger |
|---|---|
| `EPERM` | Missing `CAP_SYS_MODULE`; `kernel.modules_disabled=1`; `kernel_lockdown(integrity)` blocking unsigned. |
| `EBADF` | `fd` not a valid open fd. |
| `EINVAL` | Unknown flag bit; bad `param_values`; bad ELF / decompression; `IGNORE_MODVERSIONS`/`IGNORE_VERMAGIC` denied under hardened mode. |
| `EFAULT` | `param_values` user pointer faults. |
| `ENOEXEC` | ELF magic / arch / version mismatch. |
| `ENOMEM` | Out of memory. |
| `EEXIST` | Module with same name already loaded. |
| `EBUSY` | Concurrent load races. |
| `EKEYREJECTED` | Signature verification failed under enforced mode. |
| `ENOKEY` | Unsigned and `modules_signed_only=1`. |
| `ELIBBAD` | `__versions` CRC mismatch and `IGNORE_MODVERSIONS` not set. |
| `EBADMSG` | Compressed image decompression failed (bad zstd/xz/gz stream). |

### abi surface

```text
__NR_finit_module  (x86_64)  = 313
__NR_finit_module  (arm64)   = 273
__NR_finit_module  (riscv)   = 273
__NR_finit_module  (i386)    = 350

/* fd must be a regular-file fd or a memfd; pipe/socket/dir → EINVAL. */
/* Compressed image is decompressed in-kernel using the compressor
   selected by file magic (zstd/xz/gz); cap MODULE_INIT_LIMIT applies
   to decompressed size, not on-disk size. */
```

### compatibility contract

REQ-1: Syscall number is **313** on x86_64. ABI-stable since Linux 3.8.

REQ-2: Caller MUST have `CAP_SYS_MODULE` in init_user_ns; capability check precedes any work.

REQ-3: `flags & ~(IGNORE_MODVERSIONS | IGNORE_VERMAGIC | COMPRESSED_FILE)` non-zero → `-EINVAL`.

REQ-4: `fd` validated: regular file or memfd; otherwise `-EINVAL`.

REQ-5: If `COMPRESSED_FILE`: kernel reads the entire file into kvmalloc'd scratch, selects decompressor from magic, decompresses to image buffer. Decompressed size MUST NOT exceed `MODULE_INIT_LIMIT` (default 64 MiB).

REQ-6: Signature verification: same as `init_module(2)`:
- If `CONFIG_MODULE_SIG_FORCE=y` or sysctl `modules_signed_only=1`: image MUST carry valid appended signature → else `-EKEYREJECTED` / `-ENOKEY`.
- Signature is computed over the decompressed image, not the on-disk file.

REQ-7: IMA / IPE integrity hook: kernel calls `security_kernel_read_file(file, READING_MODULE)`. If IMA policy denies (bad hash, missing measurement), load fails with `-EACCES` or `-EPERM` per LSM.

REQ-8: `MODULE_INIT_IGNORE_MODVERSIONS`: bypass per-imported-symbol CRC check. Under hardened module loader this flag is rejected (`-EINVAL`).

REQ-9: `MODULE_INIT_IGNORE_VERMAGIC`: bypass kernel vermagic check (e.g. `7.1.0-rc2 SMP preempt mod_unload`). Under hardened module loader this flag is rejected.

REQ-10: ELF parse stages, layout, relocation, param-apply, init(), publish — identical to `init_module(2)` post-image-prep.

REQ-11: Idempotent-load coalesce (`idempotent_init_module`): kmod / systemd can call finit_module concurrently for the same module name; kernel coalesces, second caller waits, observes the first caller's success/failure return.

REQ-12: `fd`'s file mode must include FMODE_READ; if O_WRONLY or O_PATH, `-EBADF`.

REQ-13: `kernel_lockdown(integrity)` and `kernel_lockdown(confidentiality)` block `IGNORE_MODVERSIONS` and `IGNORE_VERMAGIC` unconditionally.

REQ-14: `kernel.modules_disabled=1` blocks all finit_module calls (terminal until reboot).

REQ-15: Successful load: `__init` section freed; module visible at `/proc/modules` and `/sys/module/<name>/`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_checked_first` | INVARIANT | EPERM before any fdget/copy. |
| `flag_bits_validated` | INVARIANT | unknown bit ⟹ EINVAL pre-work. |
| `compressed_size_bounded` | INVARIANT | decompressed.len() ≤ MODULE_INIT_LIMIT. |
| `ignore_flags_blocked_in_lockdown` | INVARIANT | lockdown ⟹ IGNORE_* flags ⟹ EINVAL. |
| `signature_after_decompress` | INVARIANT | verify_signature operates on decompressed bytes. |
| `idempotent_coalesce` | INVARIANT | concurrent same-name loads return identical result. |

### Layer 2: TLA+

`kernel/module-finit.tla`:
- States: per-cap, per-fdget, per-IMA, per-read, per-decompress, per-verify, per-parse, per-publish, per-init.
- Properties:
  - `safety_lockdown_blocks_ignore_flags`.
  - `safety_decompress_bounded`.
  - `safety_idempotent_coalesce_correct`.
  - `safety_unsigned_blocked_under_signed_only`.
  - `liveness_load_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_finit_module` post: ret 0 ⟹ module in MODULE_LIST | `Module::do_finit_module` |
| `decompress` post: returned image starts with ELF magic | `Module::decompress` |
| `check_vermagic` post: mismatch ⟹ ENOEXEC | `Module::check_vermagic` |
| `idempotent_dispatch` post: coalesced waiters observe same rc | `Module::idempotent_dispatch` |

### Layer 4: Verus / Creusot functional

Per-`finit_module(2)` man-page and per-`kernel/module/main.c` semantic equivalence. LTP `finit_module*`, kmod-test-suite pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`finit_module(2)` reinforcement:

- **Per-cap-check pre-fdget** — defense against per-priv-escalation via fd-side smuggling.
- **Per-fd FMODE_READ enforced** — defense against per-O_PATH/O_WRONLY bypass.
- **Per-decompressed-size bound** — defense against per-zip-bomb OOM.
- **Per-signature on decompressed image** — defense against per-decompressor-canon-mismatch attacks.
- **Per-IMA/IPE security_kernel_read_file** — defense against per-tampered-on-disk module.
- **Per-idempotent-coalesce** — defense against per-load-race UAF.

### grsecurity / pax surface

- **PaX UDEREF on `param_values` copy_from_user** — defense against per-attr-pointer kernel-deref bug; SMAP forced.
- **CAP_SYS_MODULE strict in init_user_ns only** — child userns root denied; GRKERNSEC_CHROOT_CAPS extends this to chroot.
- **GRKERNSEC_KMOD restrictions** — finit_module disabled after late-boot latch; live module loads only in early-userspace window.
- **GRKERNSEC_MODHARDEN (require signed modules)** — `CONFIG_MODULE_SIG_FORCE` semantics forced; unsigned image → `-ENOKEY`. Signing key offline; live kernel cannot enroll new keys via `keyctl`.
- **kernel_lockdown integrity-mode block** — under `kernel_lockdown=integrity` or higher: `IGNORE_MODVERSIONS` / `IGNORE_VERMAGIC` flags unconditionally rejected (`-EINVAL`). Compressed-file load still permitted (decompressed image still signature-checked).
- **PAX_USERCOPY_HARDEN on raw file read and decompressed chunk copy** — bounded copy via whitelisted bounce buffer; no slab leak even on adversarial compressed streams.
- **PaX KERNEXEC on module text pages** — module `.text` mapped RX-only post-relocation; `.rodata` mapped RO; `.data` NX.
- **GRKERNSEC_HIDESYM on `/proc/modules` and `/sys/module/*/sections/`** — module addresses concealed from non-root, defending against per-symbol-leak gadget discovery.
- **PAX_REFCOUNT on module refcount** — defense against per-refcount-overflow UAF on rmmod/insmod stress.
- **GRKERNSEC_AUDIT_MODULES** — every finit_module logged with PID, comm, fd path (resolved via dentry), flags, signature status, return code.
- **IMA appraise enforced** — under hardened mode, `ima_policy=appraise` mandatory for `READING_MODULE`; the on-disk file's IMA xattr MUST validate before in-memory load proceeds.
- **GRKERNSEC_KMOD_LIMIT** — finit_module rate-limited per uid (default 10/min); spam triggers audit + temporary block.
- **GRKERNSEC_KSTACKOVERFLOW** — defense against per-module-init stack overflow.
- **fd path resolved via fdget(REGULAR_FILE_ONLY)** — pipe/socket/dir/anon-inode fd rejected.

