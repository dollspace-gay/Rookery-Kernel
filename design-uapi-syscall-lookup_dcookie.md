---
title: "Tier-5 syscall: lookup_dcookie(2) — syscall 212"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`lookup_dcookie(2)` resolves an opaque 64-bit **dentry cookie** (`u64 cookie`) — previously handed out by an in-kernel client of the dcookies subsystem — back to a full pathname. It was created in 2002 exclusively to support `oprofile(4)`, which streams MMAP events to userspace using cookies (instead of full paths) for compactness; the userspace daemon then calls `lookup_dcookie` to expand cookies into paths at report-build time.

Because oprofile was removed from the kernel in 5.18 and the only other consumer (perf-via-cookies) was a stillborn proposal, the entire `fs/dcookies.c` subsystem and this syscall were marked deprecated in 6.6 and **removed from the upstream syscall table in 6.7**. Rookery preserves it as an ABI-compat stub returning `-ENOSYS` for any caller, while keeping the Tier-5 doc for archaeological completeness and migration guidance.

### Acceptance Criteria

- [ ] AC-1: Default Rookery build: lookup_dcookie(any, any, any) returns -ENOSYS.
- [ ] AC-2: With CONFIG_DCOOKIES_COMPAT=y and CAP_SYS_ADMIN absent: returns -EPERM.
- [ ] AC-3: With CONFIG_DCOOKIES_COMPAT=y and zero registered issuers: returns -ENOSYS.
- [ ] AC-4: Cookie not in cache: returns -EINVAL.
- [ ] AC-5: buffer too small: returns -ENAMETOOLONG.
- [ ] AC-6: buffer user pointer invalid: returns -EFAULT.
- [ ] AC-7: Non-init-user-ns caller with CAP_SYS_ADMIN in own userns: returns -EPERM.
- [ ] AC-8: Audit subsystem records every call attempt at GRKERNSEC_AUDIT_KERN.
- [ ] AC-9: oprofile-style legacy binary on default Rookery: -ENOSYS observed; binary falls back to `/proc/<pid>/maps` parsing.
- [ ] AC-10: ABI slot 212 still reserved; no other syscall may be assigned there.

### Architecture

```rust
#[syscall(nr = 212, abi = "sysv")]
pub fn sys_lookup_dcookie(cookie: u64, buffer: UserPtr<u8>, len: usize) -> isize {
    Dcookie::do_lookup(cookie, buffer, len)
}
```

`Dcookie::do_lookup(cookie, buffer, len) -> isize`:
1. /* Rookery default: subsystem-off stub. */
2. if !cfg!(CONFIG_DCOOKIES_COMPAT) { return Err(ENOSYS); }
3. /* Cap gate: only init_user_ns CAP_SYS_ADMIN. */
4. if !current().has_cap_in(&init_user_ns(), CAP_SYS_ADMIN) {
5.     return Err(EPERM);
6. }
7. /* Lookup. */
8. let dc = Dcookie::find(cookie).ok_or(EINVAL)?;
9. /* Render path under issuer's mnt-ns. */
10. let mut scratch = PathBuf::alloc(PATH_MAX)?;       // ENOMEM
11. let n = Dcookie::render_path(&dc, &mut scratch);
12. if n + 1 > len { return Err(ENAMETOOLONG); }
13. unsafe { buffer.copy_out_bytes(scratch.as_bytes(), n)?; }   // EFAULT
14. /* Audit. */
15. audit_log!("lookup_dcookie pid={} uid={} cookie={:#x} path={:?}", current().pid, current().uid, cookie, scratch);
16. Ok(n as isize)

`Dcookie::find(cookie) -> Option<Arc<Dcookie>>`:
1. let bkt = (cookie as usize) % DCOOKIE_HASH_BUCKETS;
2. dcookie_hash[bkt].read().iter().find(|d| d.cookie == cookie).cloned()

### Out of Scope

- oprofile subsystem (removed upstream 5.18).
- d_path() rendering internals (covered in Tier-3 `fs/d_path.md`).
- `/proc/<pid>/maps` MMAP enumeration alternative (covered in Tier-5 procfs docs).
- Implementation code.

### signature

```c
int lookup_dcookie(u64 cookie, char *buffer, size_t len);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `cookie` | `u64` | in | Opaque cookie previously issued by the kernel (oprofile MMAP). |
| `buffer` | `char *` | out | User buffer to receive the resolved pathname (NUL-terminated). |
| `len` | `size_t` | in | Size of `buffer`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Length of the path written (not counting trailing NUL). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Subsystem disabled (Rookery default; upstream-6.7+). |
| `EPERM`  | Caller lacks CAP_SYS_ADMIN. |
| `EINVAL` | `cookie` does not refer to a live dcookie. |
| `ENAMETOOLONG` | Resolved pathname does not fit in `len`. |
| `ENOMEM` | Allocation for path-building scratch failed. |
| `EFAULT` | `buffer` user pointer faults. |

### abi surface

```text
__NR_lookup_dcookie  (x86_64)  = 212
__NR_lookup_dcookie  (arm64)   = 18    (removed in 6.7; reserved slot)
__NR_lookup_dcookie  (riscv)   = 18    (removed in 6.7; reserved slot)
__NR_lookup_dcookie  (i386)    = 253

/* Upstream: removed in commit dating Sep 2023.  Rookery stub: -ENOSYS. */
/* Last legitimate consumer: oprofile (removed in 5.18).                 */
```

### compatibility contract

REQ-1: Syscall number is **212** on x86_64. Reserved-but-removed in 6.7+.

REQ-2: Rookery default behavior: return `-ENOSYS` immediately, before any argument validation. This is the upstream-deprecation-compatible stub.

REQ-3: Capability gate: even when re-enabled via `CONFIG_DCOOKIES=y` (off by default in Rookery), callers must hold `CAP_SYS_ADMIN`. dcookies expose private path mappings of arbitrary kernel objects; the cap gate is the security model.

REQ-4: When enabled, the algorithm:
- Look up `cookie` in the per-CPU dcookie hash table.
- If found: render the dentry's path via `d_path()` (similar to `/proc/self/cwd` resolution).
- Write the path to `buffer` (copy_to_user), bounded by `len`.
- Return path length.

REQ-5: Cookies are issued via the kernel-internal `get_dcookie(dentry, mnt, cookie_out)`; userspace cannot mint them. Stale cookies (issuer unregistered) return `-EINVAL`.

REQ-6: Path rendering uses the **dcookies-internal mount namespace** captured at cookie issuance time, NOT the caller's current mount namespace. This was a deliberate design choice for oprofile, but is a footgun for any modern consumer.

REQ-7: Lifetime: a cookie is held by the issuer (a kernel module); when the issuer calls `dcookie_unregister`, the cookie is invalidated. There is no userspace-visible TTL.

REQ-8: Per-namespace: callers in non-init mnt-ns get cookies rendered in the issuer's mnt-ns. This is an information-leak surface — a non-init userns CAP_SYS_ADMIN holder could see init-ns paths via cookies issued by an init-ns module. Mitigation: gate by `init_user_ns`.

REQ-9: Per-/proc/sys/fs/dcookies-cache: not exposed; cache size is static (default 16 entries per CPU). The subsystem is effectively dead code.

REQ-10: Per-deprecation timeline:
- Pre-5.18: live, used by oprofile.
- 5.18: oprofile removed; lookup_dcookie has no user.
- 6.6: marked deprecated in syscalls.h with warning.
- 6.7: removed from generic syscall table; per-arch slots become reserved.

REQ-11: Rookery-specific: even with CONFIG_DCOOKIES_COMPAT=y (off by default), the syscall returns `-EPERM` for any non-init-user-ns caller, and `-ENOSYS` if the dcookies cache has zero registered issuers.

REQ-12: ABI never silently changes behavior: an old binary calling lookup_dcookie always sees `-ENOSYS` (and never a successful path) on a default Rookery build.

REQ-13: Per-audit: every lookup_dcookie call is logged at GRKERNSEC_AUDIT_KERN with caller pid/uid and cookie value (potential exploit-probe indicator).

REQ-14: lookup_dcookie(2) never modifies kernel state.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `default_enosys` | INVARIANT | CONFIG_DCOOKIES_COMPAT off ⟹ ret == -ENOSYS. |
| `cap_sys_admin_init_ns` | INVARIANT | non-init-userns or no CAP_SYS_ADMIN ⟹ EPERM. |
| `buffer_bound` | INVARIANT | copy_out bytes ≤ len. |
| `no_user_minted_cookie` | INVARIANT | cookies never originate from userspace. |
| `enametoolong_no_partial_write` | INVARIANT | path too long ⟹ no bytes written to user. |
| `audit_logged` | INVARIANT | every call (including failure) audited. |

### Layer 2: TLA+

`fs/dcookies-syscall.tla`:
- States: per-stub, per-cap, per-find, per-render, per-copy-out, per-audit.
- Properties:
  - `safety_disabled_default` — ENOSYS when subsystem off.
  - `safety_cap_init_ns_required` — EPERM otherwise.
  - `safety_no_path_leak_across_mntns` — path rendered in issuer's mnt-ns; documented.
  - `safety_audit_every_call` — audit log every call.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_lookup` post: ret == -ENOSYS when feature off | `Dcookie::do_lookup` |
| `find` post: never returns user-minted cookie | `Dcookie::find` |
| `render_path` post: bounded by PATH_MAX | `Dcookie::render_path` |

### Layer 4: Verus / Creusot functional

Per-lookup_dcookie(2) historical Linux ≤6.6 semantics; archaeological reference only. No selftests required (subsystem deprecated).

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`lookup_dcookie(2)` reinforcement:

- **Per-ENOSYS default** — defense against per-attack-surface-resurrection. Subsystem off by default; opt-in build only.
- **Per-init_user_ns CAP_SYS_ADMIN strict** — defense against per-userns-priv-escalation via cookie cross-ns information leak.
- **Per-no-userspace-cookie-mint** — defense against per-spoofed-cookie path-info-leak.
- **Per-buffer bound** — defense against per-overread.
- **Per-ENAMETOOLONG no partial write** — defense against per-truncated-path leak.
- **Per-audit-every-call** — defense against per-stealthy-probe.
- **Per-deprecation-stub** — defense against per-stale-kernel-API drift.

### grsecurity / pax surface

- **CAP_SYS_ADMIN in init_user_ns enforced** — defense against per-userns chroot-escape via cookie-cross-mount-ns information leak. Even a CAP_SYS_ADMIN holder in a sub-userns cannot call lookup_dcookie; grsec enforces init_user_ns origin.
- **Deprecated subsystem: ENOSYS by default** — Rookery ships with `CONFIG_DCOOKIES_COMPAT=n`; any caller gets ENOSYS without touching the cookie hash. Defense against per-attack-surface-resurrection.
- **PaX UDEREF on buffer copy_to_user** — defense against per-userptr kernel-deref; SMAP forced; bounded by `len`.
- **PAX_USERCOPY_HARDEN on path copy** — bounded by min(path_len, len); whitelisted slab; defense against per-overread.
- **GRKERNSEC_AUDIT_KERN every call** — every successful and failing call audited with cookie value (probe-detector).
- **PAX_REFCOUNT on Arc<Dcookie>** — defense against per-cookie-refcount UAF.
- **GRKERNSEC_HIDESYM on dcookie_hash table** — symbol not in kallsyms.
- **PAX_RANDKSTACK** — defense against per-stack-layout deduction during path rendering.
- **GRKERNSEC_CHROOT** — chrooted callers cannot reach the cookie hash; redundant defense atop the CAP+init_user_ns gate.
- **No-user-minted-cookies** — defense against per-spoofed-cookie path-info-leak; cookies are kernel-internal and unforgeable from userspace.

