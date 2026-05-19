---
title: "Tier-5 syscall: setdomainname(2) — syscall 171"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`setdomainname(2)` writes the system NIS / YP "domain name" — the `domainname` field of `struct utsname` returned by `uname(2)` — into the calling task's UTS namespace. The "domain name" set here is NOT the DNS domain (which is part of the FQDN hostname); it is the historic Sun YP / NIS+ domain identifier used by `ypbind`. Modern userland rarely sets it for anything beyond cosmetic completeness, but the syscall remains for ABI compatibility and is exposed via `/proc/sys/kernel/domainname`.

Like `sethostname(2)`, the write goes into the calling task's UTS namespace and is bounded to `__NEW_UTS_LEN = 64` bytes. Setting requires `CAP_SYS_ADMIN` in the user namespace owning the UTS namespace. Container hostname/domain isolation works identically.

Critical for: legacy NIS/YP clients, `domainname(1)` utility, and CRIU snapshot/restore of `utsname`.

This Tier-5 covers `setdomainname(2)` on x86_64; the syscall is generic-unistd and identical on all architectures.

### Acceptance Criteria

- [ ] AC-1: As root, `setdomainname("nis.example.com", 15)` returns 0; `uname(2)` reports `domainname == "nis.example.com"`.
- [ ] AC-2: As non-root in init_user_ns sharing init_uts_ns, `setdomainname("x", 1)` returns `-EPERM`.
- [ ] AC-3: As root, `setdomainname(name, 65)` returns `-EINVAL`.
- [ ] AC-4: As root, `setdomainname(NULL, 1)` returns `-EFAULT`.
- [ ] AC-5: As root, `setdomainname(name, 0)` returns 0; `uname(2)` reports empty `domainname`.
- [ ] AC-6: After container `unshare(CLONE_NEWUTS | CLONE_NEWUSER)` and `setdomainname("ct", 2)`, host's domainname is unchanged.
- [ ] AC-7: Inside container sharing host's UTS, `setdomainname` from container-root returns `-EPERM`.
- [ ] AC-8: Concurrent `setdomainname("a",1)` and `setdomainname("b",1)` results in `domainname ∈ {"a", "b"}` — no tear.
- [ ] AC-9: Writing to `/proc/sys/kernel/domainname` is equivalent to calling `setdomainname`.
- [ ] AC-10: Setting `sethostname("h",1)` does NOT modify domainname.
- [ ] AC-11: Audit log records every successful `setdomainname`.
- [ ] AC-12: Per-LTP `setdomainname01..03` pass.

### Architecture

```rust
#[syscall(nr = 171, abi = "generic")]
pub fn sys_setdomainname(name: UserPtr<u8>, len: usize) -> isize {
    SetDomainname::dispatch(name, len)
}
```

`SetDomainname::dispatch(name, len) -> isize`:
1. /* Length bounds */
2. if len > __NEW_UTS_LEN { return Err(EINVAL); }
3. if len as isize < 0 { return Err(EINVAL); }
4. /* Capability */
5. let uts_ns = current.nsproxy.uts_ns.clone();
6. if !ns_capable(uts_ns.user_ns, CAP_SYS_ADMIN) {
7.   return Err(EPERM);
8. }
9. /* Copy from user */
10. let mut buf = [0u8; __NEW_UTS_LEN + 1];
11. if len > 0 { name.read_to_slice(&mut buf[..len])?; }
12. buf[len] = 0;
13. /* Commit */
14. let mut nodename = uts_ns.name.write();    // uts_sem
15. nodename.domainname[..len].copy_from_slice(&buf[..len]);
16. nodename.domainname[len] = 0;
17. drop(nodename);
18. /* Audit */
19. audit::log_setdomainname(current, &buf[..len]);
20. Ok(0)

### Out of Scope

- `sethostname(2)` (separate Tier-5 doc).
- `uname(2)` read side (separate Tier-5 doc).
- NIS / YP protocol (userspace).
- DNS domain (FQDN) — separate concept; managed via `nodename`.
- UTS namespace creation (Tier-3 in `kernel/nsproxy.md`).
- Implementation code.

### signature

```c
int setdomainname(const char *name, size_t len);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `name` | `const char *` | in | Pointer to domainname bytes; need NOT be NUL-terminated. |
| `len` | `size_t` | in | Number of bytes from `*name` to copy. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL`  | `len > __NEW_UTS_LEN` (i.e., len > 64); or `len > INT_MAX`. |
| `EFAULT`  | `name` not readable; or `len > 0` and `name == NULL`. |
| `EPERM`   | Lacking `CAP_SYS_ADMIN` in the user namespace owning the UTS namespace. |
| `ENOSYS`  | Compiled out (rare). |

### abi surface

```text
__NR_setdomainname = 171 (generic-unistd; x86_64, arm64, riscv, ...)
__NR_setdomainname (i386) = 121
__NR_setdomainname (x32)  = 0x40000000 | 171

__NEW_UTS_LEN     = 64
struct new_utsname.domainname[65]   /* 64 bytes + NUL */

Required capability: CAP_SYS_ADMIN
Namespace scope     : per-UTS namespace
```

### Limits

```text
len            : 0 .. 64 (anything larger ⟹ -EINVAL)
domainname     : up to 64 bytes; NUL byte always written at position [len]
content        : no kernel validation; any byte sequence accepted
```

### compatibility contract

REQ-1: Syscall number is **171** on x86_64 (generic-unistd). ABI-stable since 0.12.

REQ-2: `len > __NEW_UTS_LEN` (i.e., len > 64) returns `-EINVAL`. No partial write.

REQ-3: `len > INT_MAX` (cast to signed int negative) returns `-EINVAL`.

REQ-4: Caller MUST possess `CAP_SYS_ADMIN` in `current->nsproxy->uts_ns->user_ns`. Same scoping as `sethostname(2)`.

REQ-5: When `current->nsproxy->uts_ns == &init_uts_ns`, the cap check resolves against `init_user_ns`.

REQ-6: Bytes 0..len-1 of `*name` are copied to `uts_ns->name.domainname[0..len]`. The kernel writes `'\0'` at `domainname[len]`. `domainname[len+1..]` left as-is.

REQ-7: Per-UTS-namespace: only `current->nsproxy->uts_ns->name.domainname` is mutated. Other UTS namespaces unaffected.

REQ-8: Serialized by `uts_sem` (per-namespace rw-sem).

REQ-9: `/proc/sys/kernel/domainname`: writing calls the same handler. Reading returns the current `domainname`.

REQ-10: `len == 0` is valid: clears the domainname.

REQ-11: No content validation. NUL bytes embedded in `*name` are copied verbatim (and truncate the string for downstream readers).

REQ-12: Per-fork: child shares parent's UTS unless `CLONE_NEWUTS`. Per-execve: domainname unchanged.

REQ-13: Audit: every successful call logged with task creds and (truncated) domainname.

REQ-14: Concurrent calls in same UTS namespace serialized via `uts_sem`; no torn reads.

REQ-15: x32 / i386 / arm64 / riscv: same semantics, different syscall numbers.

REQ-16: CRIU: dump captures `domainname`; restore reissues `setdomainname` after recreating UTS namespace.

REQ-17: NIS/YP integration: `ypbind` reads the value via `uname(2).domainname` to select the YP server; setting domainname does NOT trigger any kernel-side NIS action — userspace must call `ypbind` separately.

REQ-18: Initial value: `init_uts_ns.name.domainname = "(none)"` (literal string set at kernel init); this is the value returned to `uname(2)` if `setdomainname` has never been called.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `len_bounded` | INVARIANT | per-dispatch: len ≤ 64 or EINVAL. |
| `cap_in_uts_ns_owner` | INVARIANT | per-dispatch: CAP_SYS_ADMIN checked against uts_ns->user_ns. |
| `nul_terminator_written` | INVARIANT | per-commit: domainname[len] == 0. |
| `per_ns_scope` | INVARIANT | per-commit: only current.uts_ns mutated. |
| `userptr_fault_safe` | INVARIANT | per-dispatch: bad pointer ⟹ -EFAULT, no state change. |
| `hostname_unchanged` | INVARIANT | per-commit: only `domainname` field touched. |

### Layer 2: TLA+

`kernel/setdomainname.tla`:
- States: per-UTS-ns domainname; uts_sem state; task creds.
- Properties:
  - `safety_len_bounded` — write ⟹ len ≤ 64.
  - `safety_cap_check` — write ⟹ CAP_SYS_ADMIN in uts_ns owner.
  - `safety_per_ns` — write affects exactly one UTS namespace.
  - `safety_isolated_field` — `nodename` (hostname) unchanged.
  - `liveness_terminates` — call returns errno or success.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dispatch` post: ok ⟹ uts_ns.domainname[0..len] == input | `SetDomainname::dispatch` |
| `dispatch` post: ok ⟹ domainname[len] == 0 | `SetDomainname::dispatch` |
| `dispatch` post: err ⟹ uts_ns unchanged | `SetDomainname::dispatch` |
| `dispatch` post: nodename (hostname) field unchanged | `SetDomainname::dispatch` |

### Layer 4: Verus/Creusot functional

Per-`setdomainname(2)` POSIX / man-page semantic equivalence. LTP `setdomainname01..03` and `domainname(1)` smoke pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setdomainname(2)` reinforcement:

- **Per-CAP_SYS_ADMIN strict** — defense against per-non-root domainname spoofing (which could redirect NIS clients to a rogue server).
- **Per-`len ≤ 64`** — defense against per-buffer-overrun into adjacent `new_utsname` fields (especially `machine` and `release`).
- **Per-NUL termination** — defense against per-info-leak of stale bytes via `/proc/sys/kernel/domainname` reads.
- **Per-UTS-namespace scoping** — defense against per-container domainname change affecting host or other containers.
- **Per-user-namespace cap resolution** — defense against per-unprivileged-user-ns spoofing host domainname.
- **Per-uts_sem write lock** — defense against per-torn-read.
- **Per-field isolation** — defense against per-`setdomainname` accidentally clobbering `nodename`.
- **Per-audit** — defense against per-undetected NIS-redirect attack.

### grsecurity / pax surface

- **CAP_SYS_ADMIN strict in init_user_ns** — under grsec policy, `setdomainname` on init_uts_ns requires CAP_SYS_ADMIN in init_user_ns; user-ns cap-escalation rejected.
- **PaX UDEREF on `name`** — defense against per-malicious-user-pointer kernel deref bug.
- **PAX_RANDKSTACK at setdomainname entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_PROC_USERGROUP** — `/proc/sys/kernel/domainname` write permission restricted to a specific group.
- **GRKERNSEC_CHROOT_NO_SETHOSTNAME** — inside a grsec chroot, `setdomainname` (alongside `sethostname`) returns `-EPERM` unconditionally.
- **Per-grsec RBAC** — `+S` flag (sysadmin-allow) gates per-subject; default policy permits for root only.
- **Per-grsec audit** — every `setdomainname` logged with task cred, RIP, and the new domainname; audited even on success.
- **Per-grsec rate-limit** — > 10 `setdomainname` calls/sec from one UTS namespace triggers WARN + log.
- **GRKERNSEC_NIS_SAFE_DEFAULTS** — initial domainname literal `(none)` enforced; suid binaries see this value regardless of any subsequent user-space changes, preventing NIS-based suid lookups from being redirected.
- **GRKERNSEC_HIDESYM** — kernel-side domainname buffer not exposed by symbol-leak paths.

