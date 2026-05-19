# Tier-5 syscall: kexec_file_load(2) — syscall 320

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/kexec_file.c (SYSCALL_DEFINE5(kexec_file_load), kimage_file_alloc_init)
  - kernel/kexec_core.c (kimage_load_segment, machine_kexec)
  - arch/x86/kernel/kexec-bzimage64.c (bzImage parser)
  - include/uapi/linux/kexec.h (KEXEC_FILE_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (320  common  kexec_file_load)
-->

## Summary

`kexec_file_load(2)` is the modern, signature-aware kexec loader. Instead of a user-built segment table (`kexec_load(2)`), the caller supplies file descriptors for the kernel image and (optionally) an initrd, plus a command-line string. The kernel reads the image, locates an arch-specific parser (`image->fops`), verifies the appended signature against a trusted keyring, parses the image to compute segments (purgatory, kernel, initrd, cmdline, devicetree, EFI memory map), allocates and copies them into a `kimage`, and stages it for `reboot(LINUX_REBOOT_CMD_KEXEC)`.

`kexec_file_load(2)` is the only kexec loader permitted under `kernel_lockdown(integrity|confidentiality)` because it can enforce trusted-image policy in-kernel. Critical for: secure-boot chain, signed kdump capture-kernel, hypervisor live-update under attestation.

## Signature

```c
long kexec_file_load(int kernel_fd, int initrd_fd,
                     unsigned long cmdline_len, const char *cmdline,
                     unsigned long flags);
```

```c
#define KEXEC_FILE_UNLOAD       0x00000001  /* clear staged image */
#define KEXEC_FILE_ON_CRASH     0x00000002
#define KEXEC_FILE_NO_INITRAMFS 0x00000004
#define KEXEC_FILE_DEBUG        0x00000008
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `kernel_fd` | `int` | in | fd for the new kernel image (e.g. bzImage). Must be a regular file, readable. |
| `initrd_fd` | `int` | in | fd for the initramfs (or -1 if `KEXEC_FILE_NO_INITRAMFS`). |
| `cmdline_len` | `unsigned long` | in | Bytes of `cmdline` (including trailing NUL). Capped at `COMMAND_LINE_SIZE` (2048 on x86_64). |
| `cmdline` | `const char *` | in | New kernel command line (or NULL with `cmdline_len == 0`). |
| `flags` | `unsigned long` | in | Bitwise OR of `KEXEC_FILE_*`; unknown bits → `-EINVAL`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Staging succeeded (or `KEXEC_FILE_UNLOAD` cleared). |
| `-1` + `errno` | Staging failed. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Missing `CAP_SYS_BOOT`. |
| `EBADF` | `kernel_fd` (or `initrd_fd` when required) invalid; not readable; O_PATH. |
| `EINVAL` | Unknown flag; `cmdline_len > COMMAND_LINE_SIZE`; cmdline missing trailing NUL; bad image arch. |
| `EFAULT` | `cmdline` user pointer faults. |
| `EBADMSG` | Image signature missing/invalid; image parse failed; arch-handler rejects. |
| `EKEYREJECTED` | Signature key not in trusted keyring or revoked. |
| `ENOKEY` | Image unsigned and `kernel_lockdown(integrity)` or `kexec_load_disabled` requires signature. |
| `ENOMEM` | Out of memory. |
| `EBUSY` | Image already staged in target slot. |
| `ENOEXEC` | No arch fops accepts the image. |
| `EACCES` | IMA/IPE appraisal denies the file. |

## ABI surface

```text
__NR_kexec_file_load  (x86_64)  = 320
__NR_kexec_file_load  (arm64)   = 294
__NR_kexec_file_load  (riscv)   = 294
/* Not present on i386 (use kexec_load there). */

/* Image is parsed by arch-specific fops:
     x86_64: bzImage64 parser
     arm64:  Image parser
     riscv:  Image parser
     ppc64:  ELF parser */
/* Signature trailer is the appended IMA-style format on the kernel image
   itself; verified against the kernel-platform keyring (.platform) + the
   built-in keyring. */
```

## Compatibility contract

REQ-1: Syscall number is **320** on x86_64. ABI-stable since Linux 3.17.

REQ-2: Caller MUST have `CAP_SYS_BOOT` in init_user_ns. Capability check precedes any work.

REQ-3: `flags & ~(KEXEC_FILE_UNLOAD | KEXEC_FILE_ON_CRASH | KEXEC_FILE_NO_INITRAMFS | KEXEC_FILE_DEBUG)` non-zero → `-EINVAL`.

REQ-4: `KEXEC_FILE_UNLOAD`:
- Clears the slot (normal or crash, per `KEXEC_FILE_ON_CRASH`).
- All other args ignored.

REQ-5: `cmdline_len`:
- 0 if cmdline omitted.
- 1..COMMAND_LINE_SIZE if provided.
- `cmdline[cmdline_len - 1] == '\0'` required; else `-EINVAL`.

REQ-6: `kernel_fd` validation:
- `fdget(kernel_fd)`; must be regular file or memfd; FMODE_READ; else `-EBADF`.
- `security_kernel_read_file(file, READING_KEXEC_IMAGE)` invoked; IMA appraisal may deny (`-EACCES`).

REQ-7: `initrd_fd` validation:
- If `KEXEC_FILE_NO_INITRAMFS`: ignored.
- Else: same validation as kernel_fd; `security_kernel_read_file(file, READING_KEXEC_INITRAMFS)`.

REQ-8: Image read:
- Entire kernel_fd content loaded into kvmalloc'd buffer.
- Bounded by `KEXEC_FILE_SIZE_MAX` (arch-specific, typically 1 GiB).

REQ-9: Signature verification:
- If `CONFIG_KEXEC_SIG=y` or `kernel_lockdown(integrity)` active: image MUST carry a valid appended IMA-style signature against the platform/builtin keyring.
- Failure → `-EKEYREJECTED` or `-ENOKEY`.

REQ-10: Arch-specific parser:
- Walk `kexec_file_loaders[]`; first `probe()` returning success owns parse.
- Parser populates segments: purgatory, kernel, initrd, cmdline, devicetree (where applicable), EFI memory map.
- No parser accepts → `-ENOEXEC`.

REQ-11: Slot exclusivity: per `KEXEC_FILE_ON_CRASH` choice, slot must be empty; else `-EBUSY`.

REQ-12: `KEXEC_FILE_DEBUG`: emit verbose printk during parse/relocate. Requires `CAP_SYS_ADMIN` under hardened mode (otherwise ignored).

REQ-13: kimage allocated; segments copied into destination pages; purgatory relocated.

REQ-14: After successful stage: the new image is entered only on `reboot(LINUX_REBOOT_CMD_KEXEC)` or on panic (for crash slot).

REQ-15: `CONFIG_KEXEC_FILE=n` → `-ENOSYS`.

## Acceptance Criteria

- [ ] AC-1: Caller without `CAP_SYS_BOOT`: `-EPERM`.
- [ ] AC-2: `flags = KEXEC_FILE_UNLOAD`: clears normal slot, returns 0.
- [ ] AC-3: `kernel_fd = -1`: `-EBADF`.
- [ ] AC-4: `kernel_fd` pointing at /etc/hostname (not bzImage): `-ENOEXEC`.
- [ ] AC-5: `cmdline_len > COMMAND_LINE_SIZE`: `-EINVAL`.
- [ ] AC-6: `cmdline_len = 4`, cmdline not NUL-terminated: `-EINVAL`.
- [ ] AC-7: Valid signed bzImage, valid initrd: returns 0.
- [ ] AC-8: Tampered signature: `-EKEYREJECTED`.
- [ ] AC-9: Unsigned image under `kernel_lockdown(integrity)`: `-ENOKEY`.
- [ ] AC-10: IMA appraisal fails on kernel_fd: `-EACCES`.
- [ ] AC-11: `flags = KEXEC_FILE_ON_CRASH` without `crashkernel=`: `-EINVAL`.
- [ ] AC-12: Re-stage without unload: `-EBUSY`.
- [ ] AC-13: `flags = 0x10` (unknown bit): `-EINVAL`.
- [ ] AC-14: After successful stage: `reboot(LINUX_REBOOT_CMD_KEXEC)` boots new image.

## Architecture

```rust
#[syscall(nr = 320, abi = "sysv")]
pub fn sys_kexec_file_load(kfd: i32, ifd: i32, cmdline_len: usize, cmdline: UserPtr<u8>, flags: usize) -> isize {
    Kexec::do_kexec_file_load(kfd, ifd, cmdline_len, cmdline, flags)
}
```

`Kexec::do_kexec_file_load(kfd, ifd, cmd_len, cmd_ptr, flags) -> isize`:
1. Kexec::check_cap_sys_boot(&current_creds())?;          // EPERM
2. const VALID = KEXEC_FILE_UNLOAD | KEXEC_FILE_ON_CRASH | KEXEC_FILE_NO_INITRAMFS | KEXEC_FILE_DEBUG;
3. if (flags & !VALID) != 0 { return -EINVAL; }
4. let crash = (flags & KEXEC_FILE_ON_CRASH) != 0;
5. let slot = if crash { Slot::Crash } else { Slot::Normal };
6. let mutex = KEXEC_MUTEX.lock();
7. if (flags & KEXEC_FILE_UNLOAD) != 0 {
8.   Kexec::clear_slot(slot);
9.   return 0;
10. }
11. if crash && !crashkernel_reserved() { return -EINVAL; }
12. if Kexec::slot_occupied(slot) { return -EBUSY; }
13. if cmd_len > COMMAND_LINE_SIZE { return -EINVAL; }
14. let cmdline = if cmd_len > 0 {
15.   let buf = cmd_ptr.copy_in_vec(cmd_len)?;
16.   if buf.last() != Some(&0) { return Err(EINVAL); }
17.   buf
18. } else { Vec::new() };
19. let kfile = fdget(kfd)?;                              // EBADF
20. security_kernel_read_file(&kfile, READING_KEXEC_IMAGE)?;  // EACCES
21. let kbuf = Kexec::read_file(&kfile, KEXEC_FILE_SIZE_MAX)?;
22. Kexec::verify_signature(&kbuf)?;                      // EKEYREJECTED / ENOKEY
23. let parser = Kexec::find_arch_parser(&kbuf).ok_or(ENOEXEC)?;
24. let ibuf = if (flags & KEXEC_FILE_NO_INITRAMFS) == 0 {
25.   let ifile = fdget(ifd)?;
26.   security_kernel_read_file(&ifile, READING_KEXEC_INITRAMFS)?;
27.   Some(Kexec::read_file(&ifile, KEXEC_FILE_SIZE_MAX)?)
28. } else { None };
29. let mut kimage = Kexec::alloc_kimage_file(crash, flags)?;
30. parser.parse(&mut kimage, &kbuf, ibuf.as_deref(), &cmdline)?;
31. Kexec::machine_prepare(&kimage)?;
32. Kexec::install_slot(slot, kimage);
33. 0

`Kexec::verify_signature(buf) -> Result<()>`:
1. if cfg!(KEXEC_SIG_FORCE) || kernel_lockdown_active(INTEGRITY) {
2.   verify_pkcs7_signature(buf, .platform_keyring + .builtin_trusted_keys)
3.     .map_err(|_| EKEYREJECTED)?;
4. } else if has_appended_sig(buf) {
5.   verify_pkcs7_signature(buf, .builtin_trusted_keys)?;
6. }
7. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_checked_first` | INVARIANT | EPERM returned before fdget/copy. |
| `flag_bits_validated` | INVARIANT | unknown bits ⟹ EINVAL pre-work. |
| `signature_under_lockdown` | INVARIANT | lockdown ⟹ verify_signature must succeed. |
| `slot_exclusive` | INVARIANT | per-slot occupancy: at most one staged kimage. |
| `arch_parser_found` | INVARIANT | no parser ⟹ ENOEXEC. |
| `cmdline_nul_terminated` | INVARIANT | cmdline_len > 0 ⟹ cmdline ends with NUL. |

### Layer 2: TLA+

`kernel/kexec-file-load.tla`:
- States: per-cap, per-flag-validate, per-slot, per-fdget, per-IMA, per-read, per-sig, per-parse, per-install.
- Properties:
  - `safety_lockdown_blocks_unsigned`.
  - `safety_slot_exclusive`.
  - `safety_parser_total_or_enoexec`.
  - `safety_initrd_optional_when_flag_set`.
  - `liveness_load_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_kexec_file_load` post: ret 0 ⟹ slot occupied | `Kexec::do_kexec_file_load` |
| `verify_signature` post: success ⟹ signature valid against trusted keyring | `Kexec::verify_signature` |
| `read_file` post: bytes ≤ KEXEC_FILE_SIZE_MAX | `Kexec::read_file` |
| `find_arch_parser` post: matches arch fops or None | `Kexec::find_arch_parser` |

### Layer 4: Verus / Creusot functional

Per-`kexec_file_load(2)` man-page and per-`kernel/kexec_file.c` semantic equivalence. Kexec-tools `--kexec-file-syscall` selftests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`kexec_file_load(2)` reinforcement:

- **Per-cap-check pre-fdget** — defense against per-priv-escalation via fd smuggling.
- **Per-fd FMODE_READ enforced** — defense against per-O_PATH bypass.
- **Per-IMA / IPE appraisal** — defense against per-tampered-on-disk image.
- **Per-signature mandatory under lockdown** — defense against per-unsigned image staging.
- **Per-slot exclusive** — defense against per-double-stage UAF.
- **Per-cmdline NUL-terminated** — defense against per-cmdline-overrun.

## Grsecurity / PaX surface

- **PaX UDEREF on `cmdline` copy_from_user** — defense against per-cmdline-pointer kernel-deref bug; SMAP forced for `cmdline_len` bytes.
- **CAP_SYS_BOOT strict in init_user_ns only** — child userns root denied; GRKERNSEC_CHROOT_CAPS extends this to chroot.
- **kernel_lockdown integrity-mode block** — under `kernel_lockdown=integrity` (or `=confidentiality`), `kexec_file_load(2)` is the **only** permitted kexec loader, and signature verification is mandatory; unsigned image → `-ENOKEY`.
- **GRKERNSEC_KEXEC_SIG_FORCE** — `CONFIG_KEXEC_SIG_FORCE` semantics forced regardless of build: every staged image MUST verify against `.platform_keyring + .builtin_trusted_keys`. The platform keyring is populated only by firmware (UEFI db/dbx); live additions denied.
- **GRKERNSEC_KEXEC_FILE_ONLY** — `kexec_load(2)` is permanently disabled; only `kexec_file_load(2)` permitted. Defense against unsigned legacy-loader use.
- **PAX_USERCOPY_HARDEN on file read into kvmalloc** — bounded copy via whitelisted bounce buffer.
- **PaX KERNEXEC on destination pages** — staged segment pages mapped RW kernel-side; only become executable via the verified purgatory + relocate_kernel trampoline.
- **IMA appraise enforced** — `ima_policy=appraise` mandatory for `READING_KEXEC_IMAGE` and `READING_KEXEC_INITRAMFS`; both files' IMA xattrs must validate.
- **GRKERNSEC_HIDESYM on `/proc/iomem` and `crashkernel` region** — defense against per-region-leak.
- **PAX_REFCOUNT on kimage refcount** — defense against per-refcount-overflow UAF.
- **GRKERNSEC_AUDIT_KEXEC** — every kexec_file_load logged with PID, comm, kernel_fd path (via dentry), initrd_fd path, cmdline (redacted), signature key serial, return code; staged image hash recorded.
- **GRKERNSEC_KMOD restrictions correlation** — if `modules_disabled=1`, kexec_file_load also enforces signature even if `KEXEC_SIG=n` (cross-restriction).
- **CAP_SYS_RAWIO not implied** — kexec_file_load requires `CAP_SYS_BOOT`; calling without it returns `-EPERM`.
- **GRKERNSEC_PERF_HARDEN correlation** — kexec staging logged under perf-audit; staging during high-rate perf flagged.
- **Crash-image hash verified at panic** — under hardened mode, staged crash kimage hash is re-verified before being entered from panic(); mismatch → fall back to plain panic.
- **KEXEC_FILE_DEBUG requires CAP_SYS_ADMIN** — verbose printk during parse gated.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Arch-specific bzImage / Image parsers (covered in arch Tier-3 docs).
- IMA policy engine (covered in Tier-3 `security/integrity/ima.md`).
- Platform keyring management (covered in Tier-3 `security/keys/platform.md`).
- Implementation code.
