# Tier-5 syscall: uname(2) — syscall 63

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE1(newuname), do_newuname)
  - include/uapi/linux/utsname.h (struct new_utsname, struct utsname)
  - kernel/utsname.c (uts_namespace)
  - arch/x86/entry/syscalls/syscall_64.tbl (63  common  uname)
-->

## Summary

`uname(2)` returns identification of the running kernel via a fixed-layout `struct utsname` with six NUL-terminated character fields: `sysname`, `nodename`, `release`, `version`, `machine`, and (Linux extension) `domainname`. Each field is `_UTSNAME_LENGTH` (65 bytes on Linux, including NUL). The data is sourced from the `uts_namespace` attached to the calling task; per-utsns isolation lets containers report a different hostname/release.

Historically there were three syscalls: `uname` (oldest, fields 9 bytes), `olduname` (compat), and `newuname` (the canonical modern entry). Linux maps the modern `uname(2)` to `newuname` via syscall 63 on x86_64.

Critical for: every `gnu-config`-style autoconf probe, `getconf(1)`, container runtime injection of `nodename`/`release` for image compatibility, `personality(2)` interaction.

## Signature

```c
int uname(struct utsname *buf);
```

```c
#define _UTSNAME_LENGTH        65
#define _UTSNAME_DOMAIN_LENGTH 65

struct new_utsname {
    char sysname    [_UTSNAME_LENGTH];   /* e.g. "Linux" */
    char nodename   [_UTSNAME_LENGTH];   /* host name (sethostname(2)) */
    char release    [_UTSNAME_LENGTH];   /* e.g. "7.1.0-rc2" */
    char version    [_UTSNAME_LENGTH];   /* e.g. "#1 SMP PREEMPT_DYNAMIC ..." */
    char machine    [_UTSNAME_LENGTH];   /* e.g. "x86_64" */
    char domainname [_UTSNAME_LENGTH];   /* NIS domain (setdomainname(2)) */
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `buf` | `struct utsname *` | out | Caller buffer; kernel writes 6 × 65 = 390 bytes of NUL-terminated strings. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `buf` user pointer faults during copy_to_user. |

## ABI surface

```text
__NR_uname (x86_64) =  63
__NR_uname (arm64)  = 160
__NR_uname (riscv)  = 160
__NR_uname (i386)   = 122  (sys_newuname)

/* Legacy x86_64 has no `olduname` / `oldolduname` (those are i386-only).
   struct new_utsname size = 6 * 65 = 390 bytes. */
```

## Compatibility contract

REQ-1: Syscall number is **63** on x86_64. ABI-stable.

REQ-2: Fields filled from `current->nsproxy->uts_ns->name`:
- `sysname` = "Linux" (compile-time constant, `UTS_SYSNAME`).
- `nodename` = utsns hostname (writable via `sethostname(2)`; default "localhost.localdomain").
- `release` = `UTS_RELEASE` (e.g. "7.1.0-rc2"). Per-utsns override possible.
- `version` = `UTS_VERSION` (build banner with `#N SMP ...`).
- `machine` = `UTS_MACHINE` (arch + variant, e.g. "x86_64").
- `domainname` = utsns NIS domain (writable via `setdomainname(2)`).

REQ-3: All fields are NUL-terminated; trailing bytes are zero (no info-leak).

REQ-4: Per-utsns: distinct utsns (CLONE_NEWUTS or unshare(2)) yields independent `nodename`/`domainname`; `sysname`/`release`/`version`/`machine` are shared globally in the upstream kernel.

REQ-5: Per-`personality(2)`: `UNAME26` personality forces `release` to be reported as "2.6.X" with X derived from kernel version (compatibility shim for ancient userspace).

REQ-6: Per-locking: `down_read(&uts_sem)` (rwsem) held during the field reads to serialize against `sethostname/setdomainname`.

REQ-7: Per-`copy_to_user`: kernel copies the whole `struct new_utsname` (390 B) in one shot.

REQ-8: Per-32-bit compat: `compat_sys_newuname` uses the same struct layout.

REQ-9: Per-old-uname (i386 only): `__NR_olduname` (109) and `__NR_oldolduname` (59) exist with shorter field lengths (9 bytes) for ancient binaries; not present on x86_64.

REQ-10: Per-`utsname()` helper inside kernel returns `current->nsproxy->uts_ns->name`; concurrent change is serialized by uts_sem.

## Acceptance Criteria

- [ ] AC-1: `uname(&u)` returns 0; `strcmp(u.sysname, "Linux") == 0`.
- [ ] AC-2: `u.machine` matches the running arch identifier.
- [ ] AC-3: After `sethostname("foo", 3)`, `u.nodename == "foo"`.
- [ ] AC-4: After `unshare(CLONE_NEWUTS); sethostname("bar", 3)` in child, parent still sees pre-existing hostname.
- [ ] AC-5: With UNAME26 personality, `u.release` reports "2.6.X".
- [ ] AC-6: `buf = NULL` → `-EFAULT`.
- [ ] AC-7: Trailing bytes within each field are zero (no stale-stack bytes).
- [ ] AC-8: Six fields each NUL-terminated.
- [ ] AC-9: Concurrent `sethostname` during `uname` does not observe a torn read.

## Architecture

```rust
#[syscall(nr = 63, abi = "sysv")]
pub fn sys_uname(buf: UserPtrMut<NewUtsname>) -> isize {
    Uts::do_uname(buf)
}
```

`Uts::do_uname(buf) -> isize`:
1. let uts = current_uts_namespace();
2. let _g = uts.uts_sem.read();
3. let mut out = NewUtsname::zeroed();
4. /* sysname */
5. out.sysname.copy_from(uts.name.sysname.as_bytes());
6. /* nodename */
7. out.nodename.copy_from(uts.name.nodename.as_bytes());
8. /* release (with personality override) */
9. if current().personality & UNAME26 != 0 {
   - out.release.copy_from(Uts::override_uname26(&uts.name.release).as_bytes());
   } else {
   - out.release.copy_from(uts.name.release.as_bytes());
   };
10. out.version.copy_from(uts.name.version.as_bytes());
11. out.machine.copy_from(uts.name.machine.as_bytes());
12. out.domainname.copy_from(uts.name.domainname.as_bytes());
13. drop(_g);
14. buf.copy_out(&out)?;                              // EFAULT
15. 0

`Uts::override_uname26(release) -> String<65>`:
1. /* Parse "X.Y.Z..." into u32 major/minor/patch */
2. let v = parse_version_triple(release);
3. /* Compute UNAME26 release: 2.6.(40 + minor) */
4. format!("2.6.{}", 40 + v.minor)

`Uts::sethostname(name, len) -> isize`:
1. require_capable(CAP_SYS_ADMIN)?;
2. if len > _UTSNAME_LENGTH - 1 { return -EINVAL; }
3. let mut buf = [0u8; _UTSNAME_LENGTH];
4. copy_from_user(&mut buf[..len], name)?;
5. let uts = current_uts_namespace();
6. let _g = uts.uts_sem.write();
7. uts.name.nodename.copy_from_slice(&buf);
8. 0

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `field_nul_terminated` | INVARIANT | every field has a NUL within _UTSNAME_LENGTH. |
| `no_stack_leak_in_padding` | INVARIANT | trailing bytes in each field are zero. |
| `uts_sem_held_for_read` | INVARIANT | uts_sem read-side held during field reads. |
| `personality_uname26_applied` | INVARIANT | UNAME26 personality ⟹ release rewritten. |
| `utsns_correctly_resolved` | INVARIANT | uses current task's uts_ns, not parent's. |

### Layer 2: TLA+

`kernel/uname.tla`:
- States: per-call namespace-resolve, per-field copy, copy_out.
- Properties:
  - `safety_no_torn_read` — concurrent sethostname / uname: uname sees old-or-new, not torn.
  - `safety_utsns_isolation` — separate utsns: hostnames don't leak.
  - `safety_no_info_leak` — fields zero-padded.
  - `liveness_terminates` — call always returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_uname` post: returns 0 ∨ -EFAULT | `Uts::do_uname` |
| `override_uname26` post: result starts with "2.6." | `Uts::override_uname26` |
| `sethostname` pre: CAP_SYS_ADMIN held | `Uts::sethostname` |

### Layer 4: Verus / Creusot functional

Per-`uname(2)` man-page semantic equivalence + per-`personality(2) UNAME26` compatibility. Selftests: `tools/testing/selftests/uts_ns/` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`uname(2)` reinforcement:

- **Per-`uts_sem` read-side held** — defense against per-torn-read race vs sethostname.
- **Per-field NUL-terminated + zero-padded** — defense against per-stack info-leak.
- **Per-utsns isolation** — defense against per-container hostname-leak.
- **Per-`copy_to_user` all-or-nothing** — defense against per-partial-write info-leak.
- **Per-`CAP_SYS_ADMIN` on sethostname** — defense against per-unprivileged hostname forge.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `buf` copy_to_user** — defense against per-user-pointer kernel-deref; SMAP forced.
- **GRKERNSEC_PROC_GETPID** — for unprivileged callers under hardened policy, `nodename` and `domainname` may be quantized/masked (e.g. replaced with "localhost" / "(none)") when the caller is in a non-init userns whose admin has not granted hostname visibility; defense against per-host-fingerprinting and per-container hostname-leak info-disclosure.
- **GRKERNSEC_HIDESYM on `version` string** — the `version` field is truncated under hardened policy: build-time strings such as `#1 SMP PREEMPT_DYNAMIC Thu Mar 7 12:34:56 UTC 2026` are reduced to a static "#1" + sanitized timestamp, removing build-host name and exact build epoch. Defense against per-kallsym-context fingerprinting and per-exploit-chain version-matching.
- **GRKERNSEC_NO_GETPID hostname leak guard** — `nodename` returned from `uname` for callers in a userns that the host has marked "hide-hostname" is replaced with a fixed token (e.g. "container"); defense against per-host-name leak across container boundaries even when `/proc/sys/kernel/hostname` is hidden.
- **Per-`UNAME26` personality access-control** — switching to UNAME26 personality is unprivileged but the rewrite is logged under grsec audit; defense against per-stealthy-personality switch covering exploit reconnaissance.
- **PAX_USERCOPY_HARDEN on `struct new_utsname` copy_to_user** — bounded 390-byte copy uses whitelisted slab region; defense against per-overlong-copy.
- **Per-`release` truncated under hardened policy** — the release string is truncated to the major-minor (e.g. "7.1") for unprivileged callers; defense against per-CVE-fingerprinting that drives exploit selection.
- **PaX KERNEXEC on uname helper inlining** — defense against per-W^X violation in the copy path.
- **GRKERNSEC_PROC_GETPID consistency with `/proc/sys/kernel/{ostype,osrelease,version,hostname,domainname}`** — sysctl shadow mirrors uname filtering rules; defense against per-bypass via sysctl.
- **PaX PAX_REFCOUNT on `nsproxy->uts_ns` refcount** — defense against per-utsns UAF during concurrent unshare/exit.
- **Per-`machine` field consistency under hardened policy** — never leaks micro-arch (e.g. "skylake") beyond "x86_64"; defense against per-microarchitecture-fingerprinting that drives Spectre/Meltdown gadget selection.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `sethostname(2)`, `setdomainname(2)` (covered in their own Tier-5 docs).
- `personality(2)` (covered separately).
- `/proc/sys/kernel/{ostype,osrelease,version,hostname,domainname}` (covered in proc / sysctl Tier-3).
- UTS namespace creation via `unshare(2)`/`clone(2)` (covered under namespace Tier-3).
- Legacy `olduname`/`oldolduname` on i386 (covered under compat Tier-5).
- Implementation code.
