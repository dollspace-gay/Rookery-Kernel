# Tier-5 syscall: security(2) — syscall 185

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (185  64  security)
  - kernel/sys_ni.c (COND_SYSCALL(security))
-->

## Summary

`security(2)` is an **unused x86_64 syscall slot** (number 185) reserved circa 2000 as a generic catch-all for Linux Security Module multiplexing — the original intent was a single-entry-point syscall that would dispatch to LSM-registered handlers via a `subcmd` argument. The design was abandoned in favor of the per-LSM-specific syscalls (`setxattr` for SELinux/SMACK labels, `keyctl` for the kernel keyring, `landlock_*` for Landlock, `bpf` for LSM-BPF, etc.).

The slot has **never been used by any mainline LSM**; it is permanently bound to `sys_ni_syscall` and unconditionally returns `-ENOSYS`.

Critical for: ABI-stability historical record; security audit of permanently-unimplemented syscall slots; defense against malware fingerprinting via syscall probing.

## Signature

```c
/* No libc header declares this. The historical design intent was: */

int security(unsigned int lsm_id, unsigned int subcmd, void *user_data, unsigned int datalen);

/* This signature was never standardized; no LSM ever registered against it. */
```

## Parameters

(unused — kernel returns ENOSYS before parameter parsing)

| Name | Type | Direction | Description |
|---|---|---|---|
| `lsm_id` | `unsigned int` | (unused) | (would have selected the LSM) |
| `subcmd` | `unsigned int` | (unused) | (would have been per-LSM subcommand) |
| `user_data` | `void *` | (unused) | (would have been per-subcmd payload) |
| `datalen` | `unsigned int` | (unused) | (would have been payload length) |

## Return value

| Value | Meaning |
|---|---|
| `-1` + `errno=ENOSYS` | Unconditional; this syscall is permanently unimplemented. |

## Errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always. Kernel returns immediately from `sys_ni_syscall`. |

## ABI surface

```text
__NR_security  (x86_64)   = 185

/* Not allocated on i386, arm64, riscv64, or any other architecture. */

/* Kernel binding: */
185  64  security  sys_ni_syscall
```

## Compatibility contract

REQ-1: Syscall number is **185** on x86_64 only. Not present on any other architecture.

REQ-2: The kernel handler is unconditionally `sys_ni_syscall`, returning `-ENOSYS`. No argument is read, no permission check, no LSM hook.

REQ-3: The slot has never been implemented in any mainline kernel. The original LSM-multiplex design was abandoned during the SELinux merge (2003), which chose to extend `setxattr` rather than introduce a new syscall.

REQ-4: The slot MUST remain `sys_ni_syscall`; reusing it for a different syscall would silently change behavior for any program probing the slot.

REQ-5: glibc does not provide a `syscall(__NR_security, ...)` wrapper; programs invoking it must construct the syscall manually.

REQ-6: `strace` recognizes the syscall number as `security(...)`; result is `-1 ENOSYS`.

REQ-7: Audit subsystem records syscall 185 and the ENOSYS return.

REQ-8: seccomp filters can match `__NR_security` as a sentinel.

REQ-9: kprobe / tracepoint on `sys_enter` for nr=185 fires before the ENOSYS return.

REQ-10: Modern LSM userspace API is via:
  - `setxattr("security.selinux", ...)` for SELinux.
  - `setxattr("security.SMACK64", ...)` for SMACK.
  - `setxattr("security.capability", ...)` for capability sets.
  - `keyctl(2)` for the kernel keyring (LSM-backed).
  - `landlock_create_ruleset(2)`, `landlock_add_rule(2)`, `landlock_restrict_self(2)` for Landlock.
  - `bpf(BPF_PROG_LOAD, ...)` with `BPF_PROG_TYPE_LSM` for LSM-BPF.
  - `lsm_list_modules(2)`, `lsm_get_self_attr(2)`, `lsm_set_self_attr(2)` (Linux 6.8+) — the modern multiplex.

REQ-11: Any "revival" of slot 185 would constitute an ABI break — modern LSMs use the new `lsm_*(2)` family (`__NR_lsm_get_self_attr=459`, etc.) instead.

## Acceptance Criteria

- [ ] AC-1: `syscall(__NR_security, 0, 0, 0, 0)` returns -1 with errno=ENOSYS.
- [ ] AC-2: `syscall(__NR_security, ...)` does not crash regardless of arguments.
- [ ] AC-3: strace prints `security(0, 0, NULL, 0) = -1 ENOSYS`.
- [ ] AC-4: No LSM hook fires for syscall 185.
- [ ] AC-5: No memory allocation, no copy_from_user, no permission check executes.
- [ ] AC-6: Tracing `sys_enter` for nr=185 fires once per call.
- [ ] AC-7: seccomp rule on __NR_security behaves like any other syscall match.
- [ ] AC-8: The syscall table entry is `sys_ni_syscall`.

## Architecture

```rust
#[syscall(nr = 185, abi = "sysv", never_implemented)]
pub fn sys_security(_a: u64, _b: u64, _c: u64, _d: u64) -> isize {
    Errno::ENOSYS.into_isize()
}
```

Bound to `sys_ni_syscall` identical to all other ENOSYS slots:

`Stub::ni_syscall() -> isize`:
1. /* No argument access, no allocation. */
2. /* No LSM hook. */
3. /* Optional audit record. */
4. Errno::ENOSYS.into_isize()

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_returns_enosys` | INVARIANT | every invocation returns -ENOSYS. |
| `no_argument_read` | INVARIANT | no copy_from_user fires. |
| `no_allocation` | INVARIANT | no kmalloc / vmalloc. |
| `no_lsm_hook` | INVARIANT | no security_* hook fires (despite the name!). |
| `no_side_effect` | INVARIANT | task state, mm, fd table unchanged. |

### Layer 2: TLA+

`kernel/ni-syscall.tla` (shared with all ENOSYS slots):
- States: per-entry, per-return.
- Properties:
  - `safety_no_state_change` — no kernel state mutates.
  - `safety_constant_return` — always ENOSYS.
  - `liveness_immediate` — returns in O(1).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_security` post: returns -ENOSYS | `Stub::ni_syscall` |
| `sys_security` post: no allocation, no copy | `Stub::ni_syscall` |

### Layer 4: Verus / Creusot functional

Per-`sys_ni_syscall` semantics. ABI-stable since the slot was reserved (~2000).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`security(2)` reinforcement:

- **Per-ENOSYS-strict response** — defense against per-future-revival ABI break.
- **Per-no-argument-read** — defense against per-arg-fuzz DoS.
- **Per-no-allocation** — defense against per-ENOSYS-allocator-pressure attack.
- **Per-no-LSM-hook (despite the name)** — defense against per-LSM-multiplex confusion.
- **Per-audit-attempt** — defense against per-stealth-probe by malware (the misleading "security" name is a common malware-fingerprint target).

## Grsecurity / PaX surface

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every `security(2)` invocation is audited. The slot's misleading name makes it a particularly attractive target for malware fingerprinting ("does the kernel respond to security(...)?"), so grsec audits invocations with elevated severity.
- **PaX KERNEXEC neutral** — no exec; bounded.
- **PaX UDEREF neutral** — no copy_from_user; bounded.
- **GRKERNSEC_HIDESYM** — `sys_ni_syscall` symbol not exposed via /proc/kallsyms.
- **PAX_USERCOPY_HARDEN neutral** — no usercopy.
- **GRKERNSEC_BRUTE_PROTECT** — repeated ENOSYS probes may trigger brute-force-protection signal escalation.
- **No CAP requirement** — ENOSYS is unprivileged; grsec preserves this.
- **No_new_privs neutral** — ENOSYS slot is not a privilege boundary.
- **Permanent ABI freeze** — grsec policy treats the slot as immutable.
- **seccomp interaction** — grsec respects seccomp filters on __NR_security.
- **No syscall-table mutation** — grsec verifies at boot that sys_call_table[185] == sys_ni_syscall; divergence triggers a panic in PAX_HARDENED. This is **critical** for slot 185 because a kernel rootkit that wanted to hide an "evade-LSM" hook would naturally pick the syscall named "security" for plausible-deniability — grsec specifically defends against this.
- **Cross-correlation with tuxcall (184) and afs_syscall (183)** — grsec audits sequential probing of the cluster of unimplemented x86_64 slots more aggressively.
- **Modern LSM API redirection** — grsec recommends (via warning) that any process attempting `security(2)` use the modern `lsm_*(2)` family instead.

## Historical context

The `security(2)` slot was reserved during early LSM design discussions (~2000-2001, pre-LSM-merge era). The original Linux Security Module proposal contemplated a generic, multiplexed syscall that would dispatch to LSM-registered handlers via an `lsm_id` argument, allowing LSMs to expose custom userspace operations (e.g. SELinux label query, capability negotiation) without each LSM needing its own syscall number.

This design was rejected during the LSM merge (Linux 2.5.29, 2002) for several reasons:

- **xattr-based labeling won**: SELinux, SMACK, and similar label-based MACs use `setxattr("security.<lsm-name>", ...)` and `getxattr(...)`, which fits cleanly into the existing VFS xattr API and requires no new syscall.
- **Per-LSM syscalls were preferred**: For LSMs needing dedicated syscalls (keyctl for the keyring, later landlock_*, later lsm_*), per-LSM numbering was cleaner than a multiplex.
- **Multiplex syscalls are footguns**: `ioctl`-style multiplexers (cf. `ioctl(2)`, `prctl(2)`, `bpf(2)`) require careful capability gating per subcommand; the LSM-multiplex would have inherited every bug class of those syscalls.

The slot remained reserved as a placeholder, but with no LSM-multiplex design ever revived, it has been permanently ENOSYS. The modern Linux 6.8+ `lsm_*(2)` family (`lsm_list_modules`, `lsm_get_self_attr`, `lsm_set_self_attr`, syscalls 459-461) provides a partial reconciliation: a small set of LSM-introspection syscalls, but not a generic multiplex.

## Forensic indicators

A process that issues `syscall(185, ...)` is almost certainly one of:

1. A program built against an obsolete header that defined `__NR_security` (rare; few headers ever did).
2. A static-analyzer probing ENOSYS slots.
3. **Malware**: the `security(2)` slot is an attractive target for kernel rootkits because:
   - The name "security" provides plausible deniability if a rootkit hooks the syscall table.
   - A real call to `security(2)` should be exceptionally rare; any caller is suspicious.
   - Anti-malware tools that whitelist common syscalls may overlook 185.

grsec's GRKERNSEC_DEPRECATED_SYSCALL_AUDIT specifically watches slot 185 and flags any sys_call_table[185] != sys_ni_syscall at boot — a kernel rootkit modifying this slot is one of the few rootkit signatures that is statically detectable.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- LSM framework (Tier-3 `security/lsm.md`).
- `lsm_list_modules(2)`, `lsm_get_self_attr(2)`, `lsm_set_self_attr(2)` modern API (separate Tier-5 docs).
- `setxattr(2)` (separate Tier-5 doc — current LSM label API).
- `keyctl(2)` (separate Tier-5 doc).
- `landlock_*(2)` (separate Tier-5 docs).
- `sys_ni_syscall` infrastructure (Tier-3 `kernel/sys_ni.md`).
- Implementation code.
