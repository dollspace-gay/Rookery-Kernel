# Tier-5 syscall: tuxcall(2) — syscall 184

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (184  64  tuxcall)
  - kernel/sys_ni.c (COND_SYSCALL(tuxcall))
-->

## Summary

`tuxcall(2)` is an **unimplemented x86_64 syscall slot** (number 184) reserved circa 2000 for the **TUX web server** — an in-kernel HTTP accelerator developed by Ingo Molnár for Red Hat, designed to serve static content from kernel space at high throughput. TUX was the precursor to today's user-space accelerators (nginx, lighttpd) and the in-kernel `khttpd`. Although TUX shipped in Red Hat Enterprise Linux 2.x and 3 as an out-of-tree module, it was **never merged into the mainline kernel**, and the `tuxcall` syscall slot was reserved for future merging that never happened.

Since the mainline kernel never accepted TUX, the syscall has always been bound to `sys_ni_syscall` and unconditionally returns `-ENOSYS`. The slot is permanently dead.

Critical for: ABI-stability historical record; security audit of permanently-unimplemented syscall slots; defense against malware fingerprinting via syscall probing.

## Signature

```c
/* No libc header declares this. The historical TUX out-of-tree patch defined: */

int tuxcall(unsigned int subcmd, void *user_data, unsigned int datalen);

/* This signature only ever existed in the out-of-tree TUX patch, never in mainline. */
```

## Parameters

(unused — kernel returns ENOSYS before parameter parsing)

| Name | Type | Direction | Description |
|---|---|---|---|
| `subcmd` | `unsigned int` | (unused) | (would have been TUX subcommand: register listener, query stats, etc.) |
| `user_data` | `void *` | (unused) | (would have been per-subcmd payload) |
| `datalen` | `unsigned int` | (unused) | (would have been payload length) |

## Return value

| Value | Meaning |
|---|---|
| `-1` + `errno=ENOSYS` | Unconditional; this syscall is permanently unimplemented in mainline. |

## Errors

| errno | Trigger |
|---|---|
| `ENOSYS` | Always. Kernel returns immediately from `sys_ni_syscall`. |

## ABI surface

```text
__NR_tuxcall  (x86_64)   = 184

/* Not allocated on i386, arm64, riscv64, or any other architecture. */

/* Kernel binding: */
184  64  tuxcall  sys_ni_syscall
```

## Compatibility contract

REQ-1: Syscall number is **184** on x86_64 only. Not present on any other architecture.

REQ-2: The kernel handler is unconditionally `sys_ni_syscall`, returning `-ENOSYS`. No argument is read, no permission check, no LSM hook.

REQ-3: The slot has never been implemented in any mainline kernel. The out-of-tree TUX patches did implement it (against RHEL 2.x kernels) but those patches are abandoned; current Red Hat / Fedora / RHEL kernels also return ENOSYS.

REQ-4: The slot MUST remain `sys_ni_syscall`; reusing it for a different syscall would silently change behavior for any program (or malware) probing the slot.

REQ-5: glibc does not provide a `syscall(__NR_tuxcall, ...)` wrapper; programs invoking it construct the syscall manually.

REQ-6: `strace` recognizes the syscall number as `tuxcall(...)`; result is `-1 ENOSYS`.

REQ-7: Audit subsystem records syscall 184 and the ENOSYS return.

REQ-8: seccomp filters can match `__NR_tuxcall` as a sentinel.

REQ-9: kprobe / tracepoint on `sys_enter` for nr=184 fires before the ENOSYS return.

REQ-10: Any "revival" of this slot would constitute an ABI break — even reviving for TUX-2.x would conflict with deployed software expecting ENOSYS.

REQ-11: The TUX project is functionally dead; modern in-kernel HTTP acceleration is provided by `io_uring` + `splice` + user-space accelerators (nginx) or by eBPF-based fast paths (XDP), not by a dedicated syscall.

## Acceptance Criteria

- [ ] AC-1: `syscall(__NR_tuxcall, 0, 0, 0)` returns -1 with errno=ENOSYS.
- [ ] AC-2: `syscall(__NR_tuxcall, ...)` does not crash regardless of arguments.
- [ ] AC-3: strace prints `tuxcall(0, NULL, 0) = -1 ENOSYS`.
- [ ] AC-4: No LSM hook fires for syscall 184.
- [ ] AC-5: No memory allocation, no copy_from_user, no permission check executes.
- [ ] AC-6: Tracing `sys_enter` for nr=184 fires once per call.
- [ ] AC-7: seccomp rule on __NR_tuxcall behaves like any other syscall match.
- [ ] AC-8: The syscall table entry is `sys_ni_syscall`.

## Architecture

```rust
#[syscall(nr = 184, abi = "sysv", never_implemented)]
pub fn sys_tuxcall(_a: u64, _b: u64, _c: u64) -> isize {
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
| `no_lsm_hook` | INVARIANT | no security_* hook fires. |
| `no_side_effect` | INVARIANT | task state unchanged. |

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
| `sys_tuxcall` post: returns -ENOSYS | `Stub::ni_syscall` |
| `sys_tuxcall` post: no allocation, no copy | `Stub::ni_syscall` |

### Layer 4: Verus / Creusot functional

Per-`sys_ni_syscall` semantics. ABI-stable since the slot was reserved (~2000).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`tuxcall(2)` reinforcement:

- **Per-ENOSYS-strict response** — defense against per-future-revival ABI break.
- **Per-no-argument-read** — defense against per-arg-fuzz DoS.
- **Per-no-allocation** — defense against per-ENOSYS-allocator-pressure attack.
- **Per-no-LSM-hook** — defense against per-policy-spoofing.
- **Per-audit-attempt** — defense against per-stealth-probe by malware.

## Grsecurity / PaX surface

- **GRKERNSEC_DEPRECATED_SYSCALL_AUDIT** — every tuxcall invocation is audited (PID, comm, process tree). Defense against malware fingerprinting the kernel by probing dead syscall slots. Particularly important because tuxcall was historically a real syscall in some out-of-tree builds; a probe sequence of "is tuxcall present, is it an out-of-tree kernel?" is a known reconnaissance pattern.
- **PaX KERNEXEC neutral** — no exec; bounded.
- **PaX UDEREF neutral** — no copy_from_user; bounded.
- **GRKERNSEC_HIDESYM** — `sys_ni_syscall` symbol not exposed via /proc/kallsyms.
- **PAX_USERCOPY_HARDEN neutral** — no usercopy.
- **GRKERNSEC_BRUTE_PROTECT** — repeated ENOSYS probes may trigger brute-force-protection signal escalation.
- **No CAP requirement** — ENOSYS is unprivileged; grsec preserves this but adds the audit trail.
- **No_new_privs neutral** — ENOSYS slot is not a privilege boundary.
- **Permanent ABI freeze** — grsec policy treats the slot as immutable.
- **seccomp interaction** — grsec respects seccomp filters on __NR_tuxcall.
- **No syscall-table mutation** — grsec verifies at boot that sys_call_table[184] == sys_ni_syscall; divergence triggers a panic in PAX_HARDENED.
- **Defense against out-of-tree TUX revival** — grsec policy refuses to load any out-of-tree module that attempts to install a sys_call_table[184] hook; such a hook is treated as a kernel-rootkit signature.
- **Cross-correlation with afs_syscall (183) and security (185)** — grsec audits sequential probing of the cluster of unimplemented x86_64 slots (183, 184, 185) more aggressively, since this is a known malware fingerprint.

## Historical context

TUX ("Threaded linUX webserver", a.k.a. `kHTTPd`'s successor) was an in-kernel HTTP server developed by Ingo Molnár at Red Hat circa 2000. The design moved HTTP request parsing and static-content serving into a kernel thread to eliminate user/kernel transition costs and to leverage the kernel's direct page-cache access for zero-copy `sendfile()` paths.

TUX shipped in Red Hat Enterprise Linux 2.1, 3, and (in some kernel SRPMs) 4 as an out-of-tree patch, including an integrated `tuxcall(2)` syscall at slot 184 used by the userspace `tux` daemon to register listeners, query statistics, and tune cache parameters. The mainline kernel reserved syscall number 184 to ensure TUX could be merged later, but the merge never happened — the consensus moved against in-kernel HTTP servers due to:

- Security: in-kernel HTTP parsing exposes the kernel directly to network-controlled string inputs; any parser bug becomes a kernel vulnerability.
- Maintenance burden: HTTP is a moving target (HTTP/2, HTTP/3); keeping a parser in the kernel was unsustainable.
- Userspace alternatives surpassed TUX: `sendfile(2)` + `epoll(2)` + nginx/lighttpd matched TUX's throughput without kernel risk.
- Modern XDP and eBPF fast-path techniques (introduced 2014+) deliver TUX-class performance with safer isolation.

The TUX project was effectively dead by 2005. RHEL 5 and later do not ship TUX. The `tuxcall(2)` slot has been ENOSYS in mainline since the slot was reserved, and remains so today.

## Forensic indicators

A process that issues `syscall(184, ...)` today is almost certainly one of:

1. A legacy program ported from RHEL 2.x/3 attempting to register with a TUX daemon that no longer exists.
2. A static-analyzer or system-call enumerator.
3. Malware fingerprinting the kernel — `tuxcall` is a famous "is this a custom kernel?" probe because some out-of-tree security-research kernels still implement it for research compatibility.

Case 3 is the security-relevant pattern; grsec audit records make it observable.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- TUX web server architecture (project abandoned; out of scope).
- In-kernel HTTP acceleration (modern: io_uring + splice, or eBPF XDP).
- `sys_ni_syscall` infrastructure (Tier-3 `kernel/sys_ni.md`).
- `sendfile(2)` (separate Tier-5 doc — the userspace zero-copy alternative).
- Implementation code.
