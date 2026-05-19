---
title: "Tier-5 syscall: sethostname(2) — syscall 170"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sethostname(2)` writes the system hostname (the `nodename` field of `struct utsname` returned by `uname(2)`) into the calling task's UTS namespace. The hostname is one of six fields managed by the UTS namespace abstraction: `sysname`, `nodename` (hostname), `release`, `version`, `machine`, `domainname`. Only `nodename` is settable via this syscall (`domainname` has its own `setdomainname(2)`; the rest are kernel-managed and immutable from userspace).

The kernel limits hostname length to `__NEW_UTS_LEN = 64` characters. The hostname is per-UTS-namespace: containers each have their own. Setting requires `CAP_SYS_ADMIN` in the user namespace owning the UTS namespace — meaning a container's root can change the container's hostname but cannot change the host's hostname (unless the container shares the host's UTS namespace, which is the default for non-`CLONE_NEWUTS` processes).

Critical for: container runtimes (Docker, Podman, LXC, systemd-nspawn) setting per-container hostname, `hostname(1)` utility, initscripts setting boot-time hostname, and DHCP clients updating hostname on lease.

This Tier-5 covers `sethostname(2)` on x86_64; the syscall is generic-unistd and identical on all architectures.

### Acceptance Criteria

- [ ] AC-1: As root, `sethostname("host1", 5)` returns 0; subsequent `uname(2)` reports `nodename == "host1"`.
- [ ] AC-2: As non-root in init_user_ns sharing init_uts_ns, `sethostname("x", 1)` returns `-EPERM`.
- [ ] AC-3: As root, `sethostname(name, 65)` returns `-EINVAL`.
- [ ] AC-4: As root, `sethostname(NULL, 1)` returns `-EFAULT`.
- [ ] AC-5: As root, `sethostname(name, 0)` returns 0; `uname(2)` reports empty `nodename`.
- [ ] AC-6: After container `unshare(CLONE_NEWUTS | CLONE_NEWUSER)` and `sethostname("ct", 2)`, host's hostname is unchanged.
- [ ] AC-7: Inside a container sharing host's UTS, `sethostname` from container-root returns `-EPERM`.
- [ ] AC-8: Concurrent `sethostname("a",1)` and `sethostname("b",1)` results in `nodename ∈ {"a", "b"}` — no tear.
- [ ] AC-9: Writing to `/proc/sys/kernel/hostname` is equivalent to calling `sethostname`.
- [ ] AC-10: Audit log records every successful `sethostname` with task cred and hostname.
- [ ] AC-11: Per-LTP `sethostname01..03` pass.

### Architecture

```rust
#[syscall(nr = 170, abi = "generic")]
pub fn sys_sethostname(name: UserPtr<u8>, len: usize) -> isize {
    SetHostname::dispatch(name, len)
}
```

`SetHostname::dispatch(name, len) -> isize`:
1. /* Length bounds */
2. if len > __NEW_UTS_LEN { return Err(EINVAL); }
3. if len as isize < 0 { return Err(EINVAL); }
4. /* Capability */
5. let uts_ns = current.nsproxy.uts_ns.clone();
6. if !ns_capable(uts_ns.user_ns, CAP_SYS_ADMIN) {
7.   return Err(EPERM);
8. }
9. /* Copy from user under write lock */
10. let mut buf = [0u8; __NEW_UTS_LEN + 1];
11. if len > 0 { name.read_to_slice(&mut buf[..len])?; }
12. buf[len] = 0;
13. /* Commit */
14. let mut nodename = uts_ns.name.write();    // uts_sem
15. nodename.nodename[..len].copy_from_slice(&buf[..len]);
16. nodename.nodename[len] = 0;
17. drop(nodename);
18. /* Audit */
19. audit::log_sethostname(current, &buf[..len]);
20. Ok(0)

### Out of Scope

- `setdomainname(2)` (separate Tier-5 doc).
- `uname(2)` read side (separate Tier-5 doc).
- UTS namespace creation via `unshare/clone(CLONE_NEWUTS)` (Tier-3 in `kernel/nsproxy.md`).
- DNS resolution / hostname-vs-FQDN semantics (userspace concern).
- Implementation code.

### signature

```c
int sethostname(const char *name, size_t len);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `name` | `const char *` | in | Pointer to hostname bytes; need NOT be NUL-terminated. |
| `len` | `size_t` | in | Number of bytes from `*name` to copy. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL`  | `len > __NEW_UTS_LEN` (i.e., len > 64); or `len` is a `size_t` value too large for the kernel's signed comparison (len > INT_MAX). |
| `EFAULT`  | `name` not readable; or `len > 0` and `name == NULL`. |
| `EPERM`   | Lacking `CAP_SYS_ADMIN` in the user namespace owning the UTS namespace. |
| `ENOSYS`  | Compiled out (rare). |

### abi surface

```text
__NR_sethostname = 170 (generic-unistd; x86_64, arm64, riscv, ...)
__NR_sethostname (i386) = 74
__NR_sethostname (x32)  = 0x40000000 | 170

__NEW_UTS_LEN     = 64           /* not including trailing NUL */
struct new_utsname.nodename[65]  /* 64 bytes + NUL terminator */

Required capability: CAP_SYS_ADMIN (capability bit 21)
Namespace scope     : per-UTS namespace (CLONE_NEWUTS isolates)
```

### Limits

```text
len            : 0 .. 64 (anything larger ⟹ -EINVAL)
nodename       : up to 64 bytes; NUL byte at position [len] always written
case / content : no kernel validation of bytes; any non-NUL byte sequence accepted
trailing NUL   : not required in input; kernel always NUL-terminates internal buffer
```

### compatibility contract

REQ-1: Syscall number is **170** on x86_64 (generic-unistd). ABI-stable since 0.12.

REQ-2: `len > __NEW_UTS_LEN` (i.e., len > 64) returns `-EINVAL`. No partial write.

REQ-3: `len < 0` is impossible (size_t), but a `len` exceeding `INT_MAX` is rejected as `-EINVAL` by the cast in the kernel.

REQ-4: Caller MUST possess `CAP_SYS_ADMIN` in `current->nsproxy->uts_ns->user_ns`. This is the user namespace that owns the UTS namespace, NOT necessarily `init_user_ns`. A container that creates its own UTS namespace via `unshare(CLONE_NEWUTS)` plus a private user namespace can change hostname inside the container with the container's root caps.

REQ-5: When `current->nsproxy->uts_ns == &init_uts_ns` (the host's UTS), the cap check resolves against `init_user_ns`. A container that shares the host's UTS therefore needs HOST root to set hostname.

REQ-6: Bytes 0..len-1 of `*name` are copied to `uts_ns->name.nodename[0..len]`. The kernel then writes `'\0'` at `nodename[len]`. Bytes at `nodename[len+1..]` are NOT zeroed (any prior contents remain). The trailing NUL guarantees `uname(2)` and `/proc/sys/kernel/hostname` always return a properly-terminated string.

REQ-7: Per-UTS-namespace: the write is to `current->nsproxy->uts_ns`. Other UTS namespaces are not affected.

REQ-8: The write is protected by `uts_sem` (a per-namespace rw-semaphore) for write; readers of `uname(2)` / `/proc/sys/kernel/hostname` use read-side.

REQ-9: `/proc/sys/kernel/hostname`: writing here calls the same handler. Reading returns the current `nodename`.

REQ-10: `len == 0` is valid: clears the hostname (sets `nodename[0] = '\0'`).

REQ-11: No content validation: the kernel does NOT verify that the hostname is a valid DNS label. It accepts any byte sequence including embedded spaces, NULs (which truncate the string for downstream readers), control characters, and non-ASCII. RFC compliance is a userspace concern.

REQ-12: Per-fork / per-clone: the child shares the parent's UTS namespace unless `CLONE_NEWUTS` is set. Setting hostname in either is visible to both if shared.

REQ-13: Per-execve: hostname unchanged (UTS namespace persists).

REQ-14: Audit: every successful `sethostname` logged with task creds, RIP, and (truncated) hostname.

REQ-15: x32 ABI / i386 / arm64 / riscv: same semantics, different syscall numbers.

REQ-16: Concurrency: concurrent `sethostname` from multiple tasks in the same UTS namespace is serialized via `uts_sem`. Readers see either old or new value, never a tear.

REQ-17: Container migration (CRIU): dump captures `nodename`; restore reissues `sethostname` after recreating UTS namespace.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `len_bounded` | INVARIANT | per-dispatch: len ≤ 64 or EINVAL. |
| `cap_in_uts_ns_owner` | INVARIANT | per-dispatch: CAP_SYS_ADMIN checked against uts_ns->user_ns. |
| `nul_terminator_written` | INVARIANT | per-commit: nodename[len] == 0. |
| `per_ns_scope` | INVARIANT | per-commit: only current.uts_ns mutated. |
| `userptr_fault_safe` | INVARIANT | per-dispatch: NULL/bad pointer ⟹ -EFAULT, no state change. |
| `uts_sem_held_on_write` | INVARIANT | per-commit: write lock held. |

### Layer 2: TLA+

`kernel/sethostname.tla`:
- States: per-UTS-ns nodename; uts_sem state; task creds.
- Properties:
  - `safety_len_bounded` — write ⟹ len ≤ 64.
  - `safety_cap_check` — write ⟹ CAP_SYS_ADMIN in uts_ns owner.
  - `safety_per_ns` — write affects exactly one UTS namespace.
  - `safety_atomic` — readers see pre- or post-state, never partial.
  - `liveness_terminates` — call returns errno or success.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dispatch` post: ok ⟹ uts_ns.nodename[0..len] == input | `SetHostname::dispatch` |
| `dispatch` post: ok ⟹ nodename[len] == 0 | `SetHostname::dispatch` |
| `dispatch` post: err ⟹ uts_ns unchanged | `SetHostname::dispatch` |
| `audit_emitted_on_ok` | `SetHostname::dispatch` |

### Layer 4: Verus/Creusot functional

Per-`sethostname(2)` POSIX / man-page semantic equivalence. LTP `sethostname01..03` and `hostname(1)` smoke pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sethostname(2)` reinforcement:

- **Per-CAP_SYS_RAWIO ... no, CAP_SYS_ADMIN strict** — defense against per-non-root hostname spoofing.
- **Per-`len ≤ 64`** — defense against per-buffer-overrun into adjacent `new_utsname` fields.
- **Per-NUL termination** — defense against per-info-leak of stale bytes via `/proc/sys/kernel/hostname` reads.
- **Per-UTS-namespace scoping** — defense against per-container hostname change affecting host.
- **Per-user-namespace cap resolution** — defense against per-unprivileged-user-ns spoofing host hostname.
- **Per-uts_sem write lock** — defense against per-torn-read of in-flight update.
- **Per-audit** — defense against per-undetected hostname change.

### grsecurity / pax surface

- **CAP_SYS_ADMIN strict in init_user_ns** — under grsec policy, `sethostname` on the HOST UTS namespace (init_uts_ns) requires CAP_SYS_ADMIN specifically in init_user_ns; user-ns-cap-escalation is rejected.
- **PaX UDEREF on `name`** — defense against per-malicious-user-pointer kernel deref bug.
- **PAX_RANDKSTACK at sethostname entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_PROC_USERGROUP** — `/proc/sys/kernel/hostname` write permission restricted to a specific group.
- **GRKERNSEC_CHROOT_NO_SETHOSTNAME** — inside a grsec chroot, `sethostname` returns `-EPERM` unconditionally even with CAP_SYS_ADMIN (chrooted processes cannot change host identity).
- **Per-grsec RBAC** — `+S` flag (sysadmin-allow) gates per-subject; default policy permits for root only.
- **Per-grsec audit** — every `sethostname` logged with task cred, RIP, and the new hostname (truncated to 64 bytes); audited even on success.
- **Per-grsec rate-limit** — > 10 `sethostname` calls/sec from one UTS namespace triggers WARN + log (defends against hostname-thrashing DoS).
- **GRKERNSEC_HIDESYM integration** — kernel-side hostname buffer not exposed by symbol-leak paths.

