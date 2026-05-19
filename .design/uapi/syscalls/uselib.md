# Tier-5 syscall: uselib(2) — syscall 134 (deprecated)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/exec.c (SYSCALL_DEFINE1(uselib))
  - fs/binfmt_elf.c (load_elf_library — deleted in modern kernels)
  - arch/x86/entry/syscalls/syscall_64.tbl (134  common  uselib)
-->

## Summary

`uselib(2)` is a **deprecated** syscall that loaded a shared library into the calling process's address space, bypassing the dynamic linker. It predates `ld.so` and the modern `dlopen(3)` model. The kernel `binfmt_elf` library-loader handler (`load_elf_library`) was removed from mainline; modern kernels always return `-ENOSYS`. It is retained as a syscall number for ABI stability and is considered a privilege-escalation hazard if re-enabled (it would let any process map executable text from arbitrary on-disk libraries without going through the dynamic linker's RELRO / nodelete / lazy-binding policies). Critical for: ABI stability with a.out / pre-glibc binaries (essentially nonexistent in practice), hardened ENOSYS contract, audit-trail.

## Signature

```c
int uselib(const char *library);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `library` | `const char *` | in | NUL-terminated pathname of the shared library to load. |

## Return value

| Value | Meaning |
|---|---|
| `0` | (Legacy only) Library loaded; entry text mapped into caller. |
| `-1` + `errno` | Failure. Modern kernels always fail with `-ENOSYS`. |

## Errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Modern kernels (default): syscall body always returns ENOSYS. |
| `EFAULT` | (Legacy) `library` userptr fault. |
| `ENAMETOOLONG` | (Legacy) `library` exceeds PATH_MAX. |
| `ENOENT` | (Legacy) Library path does not exist. |
| `EACCES` | (Legacy) No read+exec permission. |
| `ENOEXEC` | (Legacy) File is not a recognized library format. |
| `ETXTBSY` | (Legacy) File is open for writing. |
| `EPERM` | (Legacy / hardened) Caller lacks permission to load library (suid/sgid file restrictions). |

## ABI surface

```text
__NR_uselib (x86_64) = 134
__NR_uselib (arm64)  = N/A (not implemented)
__NR_uselib (riscv)  = N/A
__NR_uselib (i386)   = 86

/* Modern kernels: ENOSYS always (no binfmt_elf library handler registered). */
/* glibc's libdl uses mmap+dl-link, NOT uselib(). */
/* Legacy a.out N_SHLIB handler also removed. */
```

## Compatibility contract

REQ-1: Syscall number is **134** on x86_64. ABI-stable (number reserved; body returns ENOSYS).

REQ-2: Per `CONFIG_USELIB`: when disabled (default since ~4.4), returns `-ENOSYS`.

REQ-3: Legacy semantics (documented for historical completeness):
- Resolve `library` path via `user_path_at(AT_FDCWD, library, LOOKUP_FOLLOW)`.
- Permission check: file must be readable AND executable (`MAY_READ | MAY_EXEC`).
- Open via `file_open_root(..., O_RDONLY | O_LARGEFILE | __FMODE_EXEC)`.
- Locate matching `binfmt` handler with `load_shlib` method (only `binfmt_elf` provided one).
- Call `binfmt->load_shlib(file)` which mapped library text/data into current mm.

REQ-4: Per-hardened: modern kernels do NOT register any `load_shlib` binfmt handler; even if `CONFIG_USELIB=y` the syscall returns `-ENOSYS` because no handler matches. Grsec enforces ENOSYS regardless of config.

REQ-5: Per-suid: legacy implementation honored `MNT_NOSUID` / `S_ISUID` (if mount or library was nosuid/no-suid, uselib() refused or stripped privilege).

REQ-6: Per-mmap: legacy library load called `vm_mmap_pgoff` to map MAP_PRIVATE PROT_READ|PROT_EXEC text segment and MAP_PRIVATE PROT_READ|PROT_WRITE data segment.

REQ-7: Per-audit: every uselib(2) attempt audit-logged with library path, caller real-uid, result (ENOSYS expected).

REQ-8: No new code SHOULD use this syscall. Modern shared-library loading uses `mmap(2)` + `dlopen(3)` via glibc's dynamic linker.

REQ-9: Per-namespace: legacy implementation honored mount namespace and chroot via path lookup; modern ENOSYS makes namespace handling moot.

REQ-10: Per-policy: `uselib(2)` is on the SECCOMP-default DENY list for new sandbox profiles (systemd, Docker, runc, gVisor).

## Acceptance Criteria

- [ ] AC-1: Modern build `uselib("/lib/libc.so.6")`: returns `-ENOSYS`.
- [ ] AC-2: Modern build `uselib(NULL)`: returns `-ENOSYS` (failure observed before ptr fault).
- [ ] AC-3: Modern build `uselib("/nonexistent")`: returns `-ENOSYS`.
- [ ] AC-4: Audit record per call (regardless of build mode) with library path argument.
- [ ] AC-5: Hardened build (grsec / GRKERNSEC_USELIB_DENY): `-ENOSYS` always.
- [ ] AC-6: Seccomp default profile blocks uselib at filter-level (returns EPERM via seccomp before syscall body).

## Architecture

```rust
#[syscall(nr = 134, abi = "sysv")]
pub fn sys_uselib(library: UserPtr<u8>) -> isize {
    AuditLog::record_deprecated_syscall_attempt("uselib", library.as_addr());
    /* No load_shlib binfmt handler registered in modern kernels. */
    -ENOSYS as isize
}
```

Reference legacy implementation (NOT compiled into modern kernels):

```rust
fn legacy_uselib(library: UserPtr<u8>) -> isize {
    let path = UserPath::import(library)?;          // EFAULT/ENAMETOOLONG
    let p = Namei::user_path_at(AT_FDCWD, &path, LookupFlags::FOLLOW)?;
    if !inode_permission(p.dentry.d_inode, MAY_READ | MAY_EXEC).is_ok() { return -EACCES; }
    let file = File::open_via_dentry(&p, O_RDONLY | O_LARGEFILE | FMODE_EXEC)?;
    /* Find binfmt with load_shlib */
    for bf in binfmts_list().iter() {
        if let Some(load) = bf.load_shlib {
            match (load)(&file) {
                Ok(()) => return 0,
                Err(ENOEXEC) => continue,
                Err(e) => return -e as isize,
            }
        }
    }
    -ENOEXEC as isize
}
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `modern_returns_enosys` | INVARIANT | modern kernel ⟹ ENOSYS always. |
| `audit_pre_enosys` | INVARIANT | every call audited even when ENOSYS-returned. |
| `no_path_walk_in_modern` | INVARIANT | modern path: no `user_path_at` invoked. |
| `seccomp_default_deny` | INVARIANT | seccomp default profile blocks uselib pre-body. |

### Layer 2: TLA+

`fs/uselib.tla`:
- States: per-config-gate, per-handler-lookup, per-binfmt-call, per-return.
- Properties:
  - `safety_no_handler_registered` — no binfmt registers load_shlib in modern build.
  - `safety_enosys_always` — modern kernel ⟹ ENOSYS unconditionally.
  - `safety_audit_emitted` — every call attempt logged.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_uselib` post: returns ENOSYS in modern build | `sys_uselib` |
| `legacy_uselib` (reference) post: returns 0 only after successful load_shlib | `legacy_uselib` |

### Layer 4: Verus / Creusot functional

Modern semantic: ENOSYS contract. Legacy semantic: documented for ABI-historical completeness only.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`uselib(2)` reinforcement:

- **Per-modern-build ENOSYS strict** — defense against per-shlib-bypass-of-dl.
- **Per-audit-on-attempt** — defense against per-legacy-vector reconnaissance.
- **Per-seccomp default deny** — defense against per-legacy-syscall in unintended sandbox.
- **Per-no-binfmt-handler registered** — defense against per-handler-revival.
- **Per-DT_NODELETE / RELRO bypass closed** — defense against per-priv-escalation via crafted library.

## Grsecurity / PaX surface

- **GRKERNSEC_DEPRECATED_SYSCALL_ENOSYS_STRICT** — uselib(2) returns `-ENOSYS` unconditionally regardless of CONFIG_USELIB; emits an audit record (ANOM_DEPRECATED_SYSCALL) with caller comm/uid/library-ptr. Defense against per-shared-library-load bypass of the dynamic linker's policies (RELRO, nodelete, lazy-binding, IFUNC restrictions).
- **PaX UDEREF on `library` user-pointer copy** — defense against per-path-pointer kernel deref (even in audit path, the pointer addr is read with SMAP guard, not dereferenced).
- **GRKERNSEC_USELIB_DENY** — explicit toggle that, when set, enforces ENOSYS for uselib(2) even on builds that retain legacy binfmt_elf library support. Default-on in hardened builds.
- **GRKERNSEC_TPE compliance** — uselib(2) would bypass GRKERNSEC_TPE (trusted-path-execution) since it loads code outside exec(2); ENOSYS closes this hole.
- **GRKERNSEC_AUDIT_LEGACY_SYSCALL** — kernel.audit emits ANOM_LEGACY_USELIB record per call attempt (even when ENOSYS returned) with caller uid/comm/library-string (truncated to 256 bytes for audit).
- **PaX MPROTECT compliance** — would otherwise be a vector to map RX text without going through `execve`; ENOSYS closes the vector.
- **PaX RANDMMAP compliance** — uselib(2) mappings would not be ASLR-randomized in legacy implementation; ENOSYS prevents this.
- **PAX_REFCOUNT on caller mm refcount** — defense against per-mm refcount overflow UAF during library load (vestigial; ENOSYS short-circuits before refcount).
- **GRKERNSEC_LOG_RATE_LIMIT** — defense against per-ENOSYS log flood from repeated legacy probing.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family policy** — uselib(2) is co-neutralized with sysfs(2), ustat(2), getpmsg(2)/putpmsg(2), and unimplemented stubs; shared SIEM record-type.
- **PaX KERNEXEC on (vestigial) load_shlib path** — defense against per-W^X violation; verified disabled in hardened build.
- **GRKERNSEC_CHROOT consistency** — chroot'd callers would receive ENOSYS without leaking chroot state via library path resolution.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Modern shared-library loading via `mmap(2)` + `dlopen(3)` (out of kernel scope; userspace).
- Per-binfmt_elf executable loader (covered in Tier-3 `fs/binfmt_elf.md`).
- Per-`execve(2)` (covered in `execve.md` Tier-5).
- Implementation code.
